---
title: Using Hibernate LOB annotation the wrong and dangerous way with Postgres
date:  2025-08-12 21:30:00 +/-0200
categories: [DevOps, Databases]
tags: [hibernate, postgresql, lob, large object, annotation, error, dangerous]     # TAG names should always be lowercase
---

PostgreSQL gives you multiple ways to deal with large objects:

1. If you deal with not-so-enormous texts, and don't need pagination for performance, just stick with the [`text` data type](https://www.postgresql.org/docs/current/datatype-character.html), as it can handle unlimited length.

2. If you need to store binary data, you can go with a [`bytea` column](https://www.postgresql.org/docs/current/datatype-binary.html) and store up to `1GB` of information.

3. For really huge data, use the [Large Object facility](https://www.postgresql.org/docs/current/largeobjects.html) and handle files of up to `4TB`.

When you use the [Large Object facility](https://www.postgresql.org/docs/current/largeobjects.html), your content gets stored in a special manner outside your table, and you receive an ID by which you can retrieve your data. This means that the column in your table that references a LOB will actually only hold an ID.

```SQL
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    title VARCHAR(255),
    summary TEXT,
    content OID
);

INSERT INTO
  documents (title, summary, content)
VALUES
  ('Sample', 'Dummy content', lo_import('/path/to/your/file.txt'));
```

Your LOB being stored in a central place (a single system table named `pg_largeobject`), there can be multiple references to it. This means that when you delete from your table or drop it, Postgre won't delete the large objects you created, since it won't verify whether or not there are other references. If there aren't, the LOBs will become **orphaned**.

One way to solve this is to use the LO extension and [create a trigger](https://www.postgresql.org/docs/current/lo.html).

```SQL
CREATE EXTENSION IF NOT EXISTS lo;

CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    title VARCHAR(255),
    summary TEXT,
    content lo
);

CREATE TRIGGER t_content 
BEFORE UPDATE OR DELETE ON documents 
FOR EACH ROW EXECUTE FUNCTION lo_manage(content);
```

Of course, if you do indeed need more references to your large objects, you won't do this. Alternatively, you can periodically clean up using [ `vacuumlo`](https://www.postgresql.org/docs/current/vacuumlo.html). Its implementation is simple, according to the notes:

> vacuumlo works by the following method: First, vacuumlo builds a temporary table which contains all of the OIDs of the large objects in the selected database. It then scans through all columns in the database that are of type oid or lo, and removes matching entries from the temporary table. (Note: Only types with these names are considered; in particular, domains over them are not considered.) The remaining entries in the temporary table identify orphaned LOs. These are removed.


When using Hibernate, to benefit from the LOB facility, you must use the `@Lob` annotation. However, if your field is a string and you declare it as such, there is a subtle problem:


```Java
@Lob
@Column(name = "content")
private String content;
```

You might end up with the following table:

```SQL
CREATE TABLE documents (
  id SERIAL PRIMARY KEY,
  title VARCHAR(255),
  summary TEXT,
  content TEXT
);
```

That's right.  `@Lob` on a `String` field in Hibernate, maps to a `TEXT` column in Postgres, not to a large object OID. But being marked as `@Lob`, Hibernate will use the right functions and you will end up with id references in a `TEXT` column. (I suppose the correct way is to use `@Lob` on a `byte[]` or a stream, and map the column to the appropriate type in the database)

But storing numbers in a `TEXT` column is your smallest problem. While everything might work fine in terms of usage, the real snag arises when you want to clean up. Remember how `vacuumlo` works? It searches for `OID` or `lo` types. Since your column is not one of those, it won't see that you actually have rows that reference a LOB and will procede to delete **everything**.

If you want to test it yourself, use the `-n` parameter for a dry run, and compare the printed number with the total number of LOBs. In my case:

```SQL
SELECT distinct COUNT(*) FROM pg_largeobject;
>> 1597782
```

```sh
root@f86feb9eedbf:/mnt~ vacuumlo -n -v mydb
Connected to database "mydb"
Test run: no large objects will be removed!
Would remove 1597782 large objects from database "mydb".
```

**vacuumlo would have deleted all my LOBs**. Now, that's a wrong and dangerous way to use this facility.
