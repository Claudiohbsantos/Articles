# Indexes

```sql
CREATE INDEX dummytable_id_index ON dummytable (id);
```

Index creation can take a long time on large databases, and it is important to note that write operations (INSERT, UPDATE, DELETE) are blocked during index creation.

- Indexes can be created with more than 1 column


> "Multicolumn indexes should be used sparingly. In most situations, an index on a single column is sufficient and saves space and time. Indexes with more than three columns are unlikely to be helpful unless the usage of the table is extremely stylized. See also Section 11.5 for some discussion of the merits of different index configurations.

>> The planner will consider satisfying an ORDER BY specification either by scanning an available index that matches the specification, or by scanning the table in physical order and doing an explicit sort. For a query that requires scanning a large fraction of the table, an explicit sort is likely to be faster than using an index because it requires less disk I/O due to following a sequential access pattern. Indexes are more useful when only a few rows need be fetched. An important special case is ORDER BY in combination with LIMIT n: an explicit sort will have to process all the data to identify the first n rows, but if there is an index matching the ORDER BY, the first n rows can be retrieved directly, without scanning the remainder at all.

> A single index scan can only use query clauses that use the index's columns with operators of its operator class and are joined with AND. For example, given an index on (a, b) a query condition like WHERE a = 5 AND b = 6 could use the index, but a query like WHERE a = 5 OR b = 6 could not directly use the index.


> PostgreSQL automatically creates a unique index when a unique constraint or primary key is defined for a table. The index covers the columns that make up the primary key or unique constraint (a multicolumn index, if appropriate), and is the mechanism that enforces the constraint.


- Indexes can be created on the results of computations instead of the column raw values

- standard btree (multi-way balanced tree) index data structure.


>EXPLAIN SELECT * FROM tenk1;

>                          QUERY PLAN
> 
> -------------------------------------------------------------
>  Seq Scan on tenk1  (cost=0.00..458.00 rows=10000 width=244)
> -------------------------------------------------------------
> 
> These numbers are derived very straightforwardly. If you do:
> 
> -------------------------------------------------------------
> SELECT relpages, reltuples FROM pg_class WHERE relname = 'tenk1';
> -------------------------------------------------------------
> you will find that tenk1 has 358 disk pages and 10000 rows. The estimated cost is computed as (disk pages read * seq_page_cost) + (rows scanned * cpu_tuple_cost). By default, seq_page_cost is 1.0 and cpu_tuple_cost is 0.01, so the estimated cost is (358 * 1.0) + (10000 * 0.01) = 458.

