# Advanced CRUD Operations

### Transaction

If we want to ensure that a SQL query only execute if there's no error, we can use transaction. Here's how to do it:

```gleam
pub fn update_snippet_transaction(
  ctx: context.Context,
  title: option.Option(String),
  content: option.Option(String),
  id: Int,
) {
  case title, content {
    option.None, option.None ->
      Error(errors.BadRequest("missing title and content"))
    _, _ ->
      {
        use conn <- pog.transaction(ctx.db) // create a new transaction

        sql.update_snippet(id, title, content)
        |> db.exec(conn, _) // use the modified connection
        |> result.map_error(errors.DatabaseError)
        
        // error happens, transaction rolled back, otherwise execute the query
        result.try(function_that_can_err()) 
      }
      |> result.map_error(fn(err) { case err {
        pog.TransactionQueryError(err) -> errors.DatabaseError(err)
        pog.TransactionRolledBack(err) -> todo as "handle transaction error"
      }})
  }
}
```

I left handling the transaction error as an exercise.

### Optimistic Concurrency Control

When two user edit a snippet at the same time, it can cause a data race. In order to prevent it, we need to implement [optimistic locking](http://en.wikipedia.org/wiki/Optimistic_locking) based on version number. We already do that when we're writing the [update snippet](crud-operations.md#update-snippet) code.

Here's how it works:&#x20;

1. Alice and Bob edits a snippet at the same time. The version number is 1 when both make the request.
2. Due to various variables, Alice request comes first to the server. The edits is saved to the DB and the version number is now 2.
3. Bob request comes to the server but the version number is 1. The server sends 409 Conflict because it expects version 2. Bob need to make another request.

### Pagination

Currently our list snippet model take in a hard coded limit and offset. We don't want this so let's change it to take limit and offset as argument:

```gleam
pub fn list_snippets(ctx: context.Context, limit, offset) {
  case sql.get_snippets(limit, offset) |> db.query(ctx.db, _) {
    Ok(pog.Returned(0, _)) -> Error(errors.NotFound("snippet"))
    Ok(rows) ->
      Ok({
        rows.rows
        |> list.map(fn(row: sql.GetSnippets) {
          shared.Snippet(
            row.id,
            row.author,
            row.title,
            row.content,
            row.version,
            row.expires_at,
            row.updated_at,
            row.created_at,
          )
        })
      })
    Error(err) -> Error(errors.DatabaseError(err))
  }
}
```

Next we'll change our handler to take limit and offset from URI query:

```gleam
fn list_snippets(ctx: context.Context, req: wisp.Request) {
  let result = {
    let queries = wisp.get_query(req) // get query from URI
    // get the limit if available, fallback to 20 if it doesn't exist
    let limit = parse_int_query(queries, "limit", 20)
    // get the limit if available, fallback to 20 if it doesn't exist
    let offset = parse_int_query(queries, "offset", 0)

    snippets.list_snippets(ctx, limit, offset)
  }

  case result {
    Ok(snippets) -> {
      snippets
      |> json.array(shared.snippet_to_json)
      |> helpers.json_response("snippets", 200)
    }
    Error(err) -> errors.handle_error(req, err)
  }
}

// Take a list of queries, get a query from a key, then turn it into Int. If the key doesn't
// exist, use fallback.
fn parse_int_query(queries, key, fallback) {
  list.key_find(queries, key)
  |> result.unwrap("")
  |> int.parse()
  |> result.unwrap(fallback)
}
```

### Filtering



### Sorting
