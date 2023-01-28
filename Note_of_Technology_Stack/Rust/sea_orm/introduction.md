# SeaORM

SeaORM is a relational ORM to help build web services in Rust with the familiarity of dynamic languages.

- Async: Relying on SQLx, SeaORM is a new library with async support from day 1.
- Dynamic: Built upon SeaQuery, SeaORM allows build complex queries without 'fighting the ORM'.
- Testable: Use mock connections to write unit tests for your logic.
- Service Oriented: Quickly build services that join, filter, sort and paginate data in APIs.

## Integrate with Axum

1. Modify the `DATABASE_URL` var in `.env` to point to the chosen database.
2. Turn on the appropriate database feature for the chosen database in `core/Cargo.toml` (the `"sqlx-postgres",` line).
3. Execute `cargo run` to start the server.
4. Visit localhost:8000 in browser.

Run mock test on the core logic crate:
```
cd core
cargo test --features mock
```
