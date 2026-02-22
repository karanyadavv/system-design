## Test how indexing reduces the load on our databse

1. Create a table
2. Fill it with a lot of rows (important to actually see differences)
3. Use a SELECT statement to get a user data with EXPLAIN ANALYZE to view the time taken
4. Apply an index to the table, get data again and view the difference

### Use the docker compose file to run postgres locally

```
docker-compose up -d
```

### Enter into the postgres container you have created

```
docker exec -it <container-name-or-id> /bin/sh
```

### Open the PostgreSQL interactive terminal

```
psql -U postgres
```

### Create table

```
CREATE TABLE users(
  id SERIAL PRIMARY KEY,
  name VARCHAR(255),
  email VARCHAR(255),
  age INT
);
```

### Create large dataset (important)

```
INSERT INTO users(name, email, age)
SELECT
  'user' || gs,
  'user' || gs || '@test.com',
  (random() * 80)::int
FROM generate_series(1,1000000) AS gs;

```

### Measure Without Index

```
EXPLAIN ANALYZE
SELECT * FROM users WHERE age = 30;
```

### Important things to note

```
                                                        QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------
 Gather  (cost=1000.00..15792.33 rows=12500 width=37) (actual time=0.518..25.866 rows=12615 loops=1)
   Workers Planned: 2
   Workers Launched: 2
   ->  Parallel Seq Scan on users  (cost=0.00..13542.33 rows=5208 width=37) (actual time=0.023..19.985 rows=4205 loops=3)
         Filter: (age = 30)
         Rows Removed by Filter: 329128
 Planning Time: 0.058 ms
 Execution Time: 26.157 ms
(8 rows)
```

1. Seq Scan on users - means a full table scan has been done
2. Execution Time: 26.157 ms

### Add an index to age column

#### Note: Index `age` is bad candidate since many people share the same age. We are only using it to make our experiment simpler.

```
CREATE INDEX user_age on users(age);
```

### Run the query again with ANALYZE

Output:

```
                                                        QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on users  (cost=141.30..9030.58 rows=12500 width=37) (actual time=1.696..6.062 rows=12615 loops=1)
   Recheck Cond: (age = 30)
   Heap Blocks: exact=6525
   ->  Bitmap Index Scan on user_age  (cost=0.00..138.18 rows=12500 width=0) (actual time=0.889..0.889 rows=12615 loops=1)
         Index Cond: (age = 30)
 Planning Time: 0.500 ms
 Execution Time: 6.460 ms
(7 rows)
```

1. Query ran using the Index "user_age"
2. Execution time reduced from 26.157ms --> 6.460ms

### Conclusions

1. Postgres used Parallel workers to run a sequential scan (when no index defined)
2. When a index is defined it uses a Bitmap Index Scan
3. Execution time reduced from 26.157ms --> 6.460ms using an index
4. 4x improvement is noticed even when choosing a bad candidate for an index. Think about the perf gains when indexing is used correctly.
