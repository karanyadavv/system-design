## Test how we could create read replicas of our DB

### Agenda

1. Run a primary and replica postgres
2. Configure replication
3. Create a replication user
4. Insert some data into the primary db and observe it replicate onto the read replica

### Use the docker compose file to run postgres locally

```
docker-compose up -d
```

### Connect to the postgres primary container you have created

```
docker exec -it <container-name-or-id> psql -U postgres
```

_Note_:

- We could run the setup(creating the role, adding the connection, running basebackup) completely through the docker-compose.yaml file.
- But we are not doing this here, so we know exactly what happens when you want to setup a replica
- The naunces that makes all of this work.

### Create a replication user

```
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'replica_pass';
```

### Edit the pg_hba.conf file

- Since we are using bind volumes, for this particular usecase
- Making it easier for us to edit conf files locally
- You would have 2 folders created after running docker-compose in your current directory with all the data related to your containers
- Adding below line, tells the primary db to allow a connection to the replica
- Specify that this is not a normal connection, it's a `replication` with the user name `replicator`
- allowing from any IP and asking for a password
- Add the line below to the end of the `/primary/pg_hba.conf`

```
host replication replicator 0.0.0.0/0 md5
```

### Run basebackup

https://www.postgresql.org/docs/current/app-pgbasebackup.html

Stop replica container, delete the files

```
docker stop replica

rm .\replica\*
```

Now run the base backup:

The command below will work on Windows Powershell. If you are using bash, replace the backtick with backslash and you should be good

```
docker run --rm `
  --network read-replicas_default `
  -v ./replica:/var/lib/postgresql/data `
  -e PGPASSWORD=replica_pass `
  postgres `
  pg_basebackup -h primary -U replicator `
  -D /var/lib/postgresql/data -Fp -Xs -R -P
```

Start replica:

```
docker start replica
```

### Verify replication

Insert into primary:

```
CREATE TABLE users(id SERIAL PRIMARY KEY, name VARCHAR(255));
INSERT INTO users(name) VALUES('Karan');
```

Connect to replica:

```
docker exec -it replica psql -U postgres
SELECT * FROM users;
```

If it shows up, which means replication works

### Try writing to the replica:

```
INSERT INTO users(name) VALUES('Should fail');
```

Output:

```
ERROR:  cannot execute INSERT in a read-only transaction
```

### Conclusion

- This doesn't end here. The replica is using WAL and running the transaction independently.
- If the number of transaction grows, the time taken to re-run the transactions on the replica will lag behind.
- The diff between primary and replica might grow known as replication lag.

### Side notes:

How does read replica works

- There is something called WAL (Write-Ahead Logging). "Log first, apply later"
- Postgres does this to maintain logs of every transaction even before executing it. That's why `Write-Ahead`
- Instead of running these transaction on both instances(primary and replica)
- Postgres transfers these logs to the replica DB
- So that the replica can run the same transactions on it's own and in turn "replicate" the DB

Ports

- You might've noticed that both our services have `5432:5432` & `5433:5432` ports.
- The first port mapping, is exposing a port on our system to then talk to postgres
- Postgres typically runs on port `5342`
- And since, both services are seperated from each other. They can safely use the same `5432` port internally.
- `hostport:postgres-port`
