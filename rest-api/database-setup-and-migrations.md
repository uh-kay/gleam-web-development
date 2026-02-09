# Database Setup and Migrations

### Database Setup

Before we begin writing handler for our snippets, let's see how to make the data on the server persist using PostgreSQL database. I'm not going to explain how to install and setup PostgreSQL, you can refer to the many resources out there for the specific OS or use this [guide](https://www.docker.com/blog/how-to-use-the-postgres-docker-official-image/) for Docker.

I use this `docker-compose.yml` file to setup the database using Docker:

```yaml
## server/docker-compose.yml
services:
  postgres:
    image: postgres:18-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: root
      POSTGRES_PASSWORD: password
      POSTGRES_DB: pastem
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql

volumes:
  postgres_data:
```

`init.sql`

```sql
-- server/init.sql
CREATE EXTENSION IF NOT EXISTS citext;
```

Then run `docker compose up -d`, and a new database `pastem` will be created along with the `citext` extension.

### Migrations

For migrations I usually use [goose](https://github.com/pressly/goose), but feel free to use other tool you're more familiar with. Put this inside `.env`:

```bash
DATABASE_URL=postgres://root:password@localhost:5432/pastem?sslmode=disable
GOOSE_DRIVER=postgres
GOOSE_DBSTRING=postgres://root:password@localhost:5432/pastem
GOOSE_MIGRATION_DIR=./migrations
```

Inside `server` directory, run `goose create -s create_snippets_table sql`. It'll create a new empty SQL file. Inside it write this:

```sql
-- +goose Up
-- +goose StatementBegin
CREATE TABLE IF NOT EXISTS snippets (
    id bigserial primary key,
    author bigint not null,
    title varchar(255) not null,
    content text not null,
    expires_at bigint not null,
    updated_at bigint not null,
    created_at bigint not null
);
-- +goose StatementEnd

-- +goose Down
-- +goose StatementBegin
DROP TABLE IF EXISTS snippets;
-- +goose StatementEnd
```

To apply the migration, run `goose up`.

### Connecting to Database

Run `gleam add pog parrot`. `Pog` is PostgreSQL database client. Inside `server/src/server/`, make a new `context.gleam` directory and write this:

```gleam
// server/src/server/context.gleam

// create a new context type with db connection as the argument
pub type Context {
  Context(db: db.Connection)
}
```

To make database connection available on all handler, we will make a context that contains the database connection. This context will then be passed around, starting inside handle\_request function to make it available to all handlers.

```gleam
// server/src/server/router.gleam

// add context as handle_request argument
pub fn handle_request(ctx: context.Context, req: Request) -> Response {
  use req <- middleware.middleware(req)

  case wisp.path_segments(req) {
    ["api", "health"] -> health.health()
    _ -> wisp.not_found()
  }
}
```

To connect our server to the database, write this inside `server.gleam`:

```gleam
// server/src/server.gleam
pub fn main() -> Nil {
  wisp.configure_logger()

  let db_pool_name = process.new_name("db_pool") // create a new process with "db_pool" name
  let assert Ok(database_url) = envoy.get("DATABASE_URL") // get db URL from env variable
  let assert Ok(pog_config) = pog.url_config(db_pool_name, database_url) // create a new pog config
  let assert Ok(_) = pog_config |> pog.pool_size(10) |> pog.start // set pool size as 10 and start pog
  let con = pog.named_connection(db_pool_name) // get connection from process name

  let secret_key_base =
    result.unwrap(envoy.get("SECRET_KEY_BASE"), "changethis")

  let context = context.Context(con) // create a new context
  let handler = router.handle_request(context, _) // pass context into handle_request

  let assert Ok(_) =
    handler
    |> wisp_mist.handler(secret_key_base)
    |> mist.new
    |> mist.port(8000)
    |> mist.start

  process.sleep_forever()
}
```

### Generating Data Layer

If you have made a REST API before, you must know by now that the next step is creating the data/model layer. We can write that by hand using Pog, but we're not going to manually write the query and decoder in the big 2026. We're going to use parrot to generate wrapper for our query.

Install parrot: `gleam add parrot` . Then create `sql` directory inside `server/src/`. Create `snippets.sql` inside it and write:

```sql
-- server/src/sql/snippet.sql

-- name: CreateSnippet :exec
insert into snippets (author, title, content, expires_at, updated_at, created_at)
values ($1, $2, $3, $4, $5, $6);

-- name: GetSnippets :many
select id, author, title, content, expires_at, updated_at, created_at from snippets
limit $1 offset $2;

-- name: GetSnippet :one
select id, author, title, content, expires_at, updated_at, created_at
from snippets
where id = $1;

-- name: UpdateSnippet :exec
update snippets
set title = coalesce($1, title), content = coalesce($2, content)
where id = $3;

-- name: DeleteSnippet :exec
delete from snippets where id = $1;
```

Then run `gleam run -m parrot`. It'll create `sql.gleam` file inside `server/src/` . We don't need to edit anything in that file. Next we're going to make a wrapper for parrot and pog. Inside `server/src/server` directory, create `db.gleam`\` and write:

```gleam
// server/src/server/db.gleam

// Turn parrot params into pog value
pub fn parrot_to_pog(param: dev.Param) -> pog.Value {
  case param {
    dev.ParamDynamic(_) -> panic as "dynamic parameter need to be implemented"
    dev.ParamBool(x) -> pog.bool(x)
    dev.ParamFloat(x) -> pog.float(x)
    dev.ParamInt(x) -> pog.int(x)
    dev.ParamString(x) -> pog.text(x)
    dev.ParamBitArray(x) -> pog.bytea(x)
    dev.ParamList(x) -> pog.array(parrot_to_pog, x)
    dev.ParamNullable(x) -> pog.nullable(fn(a) { parrot_to_pog(a) }, x)
    dev.ParamDate(x) -> pog.calendar_date(x)
    dev.ParamTimestamp(x) -> pog.timestamp(x)
  }
}

// Wrapper for db query
pub fn query(
  db: pog.Connection,
  b: #(String, List(dev.Param), decode.Decoder(a)),
) {
  b.0
  |> pog.query()
  |> pog.returning(b.2)
  |> list.fold(b.1, _, fn(acc, param) {
    let param = parrot_to_pog(param)
    pog.parameter(acc, param)
  })
  |> pog.execute(db)
}

// Wrapper for exec query
pub fn exec(db: pog.Connection, b: #(String, List(dev.Param))) {
  b.0
  |> pog.query()
  |> list.fold(b.1, _, fn(acc, param) {
    let param = parrot_to_pog(param)
    pog.parameter(acc, param)
  })
  |> pog.execute(db)
}
```

### Further Reading

* Parrot wrappers: [https://github.com/daniellionel01/parrot/blob/main/docs/wrappers.md](https://github.com/daniellionel01/parrot/blob/main/docs/wrappers.md)
* Parrot examples: [https://github.com/daniellionel01/parrot/blob/main/docs/wrappers.md](https://github.com/daniellionel01/parrot/blob/main/docs/wrappers.md)
