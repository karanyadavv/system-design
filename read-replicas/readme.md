## Test how we could read replicas of our DB

### Side notes:

- You might've noticed that both our services have `5432:5432` & `5433:5432` ports.
- The first port mapping, is exposing a port on our system to then talk to postgres
- Postgres typically runs on port `5342`
- And since, both services are seperated from each other. They can safely use the same `5432` port internally.
- `hostport:postgres-port`
