# GvACCOUNTS
 Gv Notes Units of Value are measured in USD and sent or received via Gv Transfers Transactions Services. So overall We are faciltatiing Gv Send Gv Receive Gv Shop Gv Accounts GvStackMarket.

 

cr> CREATE TABLE parted_table (
...   id bigint,
...   title text,
...   content text,
...   width double precision,
...   day timestamp with time zone
... ) CLUSTERED BY (title) INTO 4 SHARDS PARTITIONED BY (day);
CREATE OK, 1 row affected (... sec)

This creates an empty partitioned table which is not yet backed by real partitions. Nonetheless does it behave like a normal table.

When the value to partition by references one or more Base Columns, their values must be supplied upon INSERT or COPY FROM. Often these values are computed on client side. If this is not possible, a generated column can be used to create a suitable partition value from the given values on database-side:

cr> CREATE TABLE computed_parted_table (
...   id bigint,
...   data double precision,
...   created_at timestamp with time zone,
...   month timestamp with time zone GENERATED ALWAYS AS date_trunc('month', created_at)
... ) PARTITIONED BY (month);
CREATE OK, 1 row affected (... sec)

Information schema
This table shows up in the information_schema.tables table, recognizable as partitioned table by a non null partitioned_by column (aliased as p_b here):

cr> SELECT table_schema as schema,
...   table_name,
...   number_of_shards as num_shards,
...   number_of_replicas as num_reps,
...   clustered_by as c_b,
...   partitioned_by as p_b,
...   blobs_path
... FROM information_schema.tables
... WHERE table_name='parted_table';
+--------+--------------+------------+----------+-------+---------+------------+
| schema | table_name   | num_shards | num_reps | c_b   | p_b     | blobs_path |
+--------+--------------+------------+----------+-------+---------+------------+
| doc    | parted_table |          4 |      0-1 | title | ["day"] | NULL       |
+--------+--------------+------------+----------+-------+---------+------------+
SELECT 1 row in set (... sec)

cr> SELECT table_schema as schema, table_name, column_name, data_type
... FROM information_schema.columns
... WHERE table_schema = 'doc' AND table_name = 'parted_table'
... ORDER BY table_schema, table_name, column_name;
+--------+--------------+-------------+--------------------------+
| schema | table_name   | column_name | data_type                |
+--------+--------------+-------------+--------------------------+
| doc    | parted_table | content     | text                     |
| doc    | parted_table | day         | timestamp with time zone |
| doc    | parted_table | id          | bigint                   |
| doc    | parted_table | title       | text                     |
| doc    | parted_table | width       | double precision         |
+--------+--------------+-------------+--------------------------+
SELECT 5 rows in set (... sec)

And so on.

You can get information about the partitions of a partitioned table by querying the information_schema.table_partitions table:

cr> SELECT count(*) as partition_count
... FROM information_schema.table_partitions
... WHERE table_schema = 'doc' AND table_name = 'parted_table';
+-----------------+
| partition_count |
+-----------------+
| 0               |
+-----------------+
SELECT 1 row in set (... sec)

As this table is still empty, no partitions have been created.

Insert
cr> INSERT INTO parted_table (id, title, width, day)
... VALUES (1, 'Don''t Panic', 19.5, '2014-04-08');
INSERT OK, 1 row affected (... sec)

cr> SELECT partition_ident, "values", number_of_shards
... FROM information_schema.table_partitions
... WHERE table_schema = 'doc' AND table_name = 'parted_table'
... ORDER BY partition_ident;
+--------------------------+------------------------+------------------+
| partition_ident          | values                 | number_of_shards |
+--------------------------+------------------------+------------------+
| 04732cpp6osj2d9i60o30c1g | {"day": 1396915200000} |                4 |
+--------------------------+------------------------+------------------+
SELECT 1 row in set (... sec)

On subsequent inserts with the same partition column values, no additional partition is created:

cr> INSERT INTO parted_table (id, title, width, day)
... VALUES (2, 'Time is an illusion, lunchtime doubly so', 0.7, '2014-04-08');
INSERT OK, 1 row affected (... sec)

cr> REFRESH TABLE parted_table;
REFRESH OK, 1 row affected (... sec)

cr> SELECT partition_ident, "values", number_of_shards
... FROM information_schema.table_partitions
... WHERE table_schema = 'doc' AND table_name = 'parted_table'
... ORDER BY partition_ident;
+--------------------------+------------------------+------------------+
| partition_ident          | values                 | number_of_shards |
+--------------------------+------------------------+------------------+
| 04732cpp6osj2d9i60o30c1g | {"day": 1396915200000} |                4 |
+--------------------------+------------------------+------------------+
SELECT 1 row in set (... sec)
