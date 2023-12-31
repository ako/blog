---
layout: post
title: JSON transformation - surprising performance of PostgreSQL vs Josson vs JSLT/JSLT2
---

# Json transformations - surprising performance of PostgreSQL vs Josson vs JSLT/JSLT2

A quick, very unscientific, test indicates that using Postgres as a JSON transformation service seems to be as fast as using a dedicated JSON query java library. The following compares execution times for a 350Kb JSON document transformed using [Postgresql SQL][3] (version 15.3), [Josson][1] and [JSLT][2]/[JSLT2][4]. The postgres DB runs on a docker on the same laptop, macbook pro 16 M1 Max, the tests are run on Windows 11 running in parallels.

It's interesting to see that using JSONB Postgres to transform the document is ~~faster~~ equally fast to using an in process java library to do the same. In the case of the Postgres approach, the document isn't stored in the database, instead it's provided as a parameter for the sql query. So the SQL timing includes sending the JSON to the database engine, and returning the result.

SQL used is as follows:

```sql
with   response as (select ?::jsonb as content)
select json_agg(x) as output
from   (    
            with objects as (
                select jsonb_each(content -> 'ServiceData' -> 'modelObjects') as obj
                from   response
            )
            select (obj).value ->> 'uid' as uid
            ,      (obj).value ->> 'className'    as classname
            ,      (obj).value ->> 'type'         as type
            ,      (obj).value -> 'props' -> 'query_name' -> 'dbValues' ->> 0   as query_name
            ,      (obj).value -> 'props' -> 'query_desc' -> 'dbValues' ->> 0   as query_desc
            ,      (obj).value -> 'props' -> 'object_string' -> 'dbValues' ->> 0   as object_string
            ,      (obj).value -> 'props' -> 'change_date' -> 'dbValues' ->> 0   as change_date
            from   objects
            order  by 4
        ) as x
```

The SQL is more complex than it needs to be, but that enables me to replace the inner from dynamically.

The Josson template looks like this:

```java
ServiceData.modelObjects.**.map
(   uid,className,type
    ,query_name:props.query_name.dbValues[0]
    ,query_desc:props.query_desc.dbValues[0]
    ,creation_date:props.creation_date.dbValues[0]
    ,object_string:props.object_string.dbValues[0]
).sort(query_name)
```

The JSLT template doesn't include the sorting part, as that's not supported by JSTL as far as i can tell:

```java
[for (.ServiceData.modelObjects) {
    "uid":.value.uid,
    "classname":.value.className,
    "type":.value.type,
    "query_name":.value.props.query_name.dbValues[0],
    "query_desc":.value.props.query_desc.dbValues[0],
    "creation_date":.value.props.creation_date.dbValues[0],
    "object_string":.value.props.object_string.dbValues[0]
}]
```

Updated results running the tests with [Microsoft JDK11 for Arm](5):

| Approach | time parsing source / binding | time transforming | Total |
|----------|-------------------------------|-------------------|-------|
| Josson   |                            16 |               156 |   172 |
| JSLT     |                            31 |                31 |    62 |
| JSLT2    |                            31 |                32 |    63 |
| SQL      |                             0 |                63 |    63 |

Results (Eclipse Temurin JDK 11):

| Approach | time parsing source / binding | time transforming | Total |
|----------|-------------------------------|-------------------|-------|
| Josson   |                           681 |              1226 |  1907 |
| JSLT     |                           120 |                94 |   214 |
| JSLT2    |                            95 |                78 |   173 |
| SQL      |                            13 |                53 |    66 |

Here's a screenshot of these 3 options:

![JSON transformation comparisson](/blog/assets/2023-08-25-json-transformation-pgsql-josson-jslt.png)

As said in the begining, this is just a quick test, so there may be ways to optimize these approaches. I've rerun the tests a couple of times to ensure the results don't include exceptional results, but they're pretty much repeatable. 

[1]: https://github.com/octomix/josson
[2]: https://github.com/schibsted/jslt
[3]: https://www.postgresql.org/docs/current/functions-json.html
[4]: https://github.com/tonysparks/jslt2
[5]: https://learn.microsoft.com/en-us/java/openjdk/download