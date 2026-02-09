# CRUD Operations

### Create Snippet

We can use the generated SQL query directly inside our handler, but currently it's not mapped to our shared Snippet type. So instead of doing that inside our handler let's do it inside our model layer.&#x20;

`$ cd server/src/server && mkdir model`

`$ cd model && touch snippets.gleam`<br>

```gleam
// server/src/server/models/snippet.gleam
pub fn create_snippet(
  ctx: context.Context,
  title: String,
  content: String,
  expires_at: timestamp.Timestamp,
) {
  timestamp.to_unix_seconds_and_nanoseconds(expires_at).0
  |> sql.create_snippet(
    1, // we hardcode the user id for now
    title,
    content,
    _,
    helpers.current_time(),
    helpers.current_time(),
  )
  |> db.exec(ctx.db, _)
  |> result.map_error(errors.DatabaseError)
}
```

Create a new `errors.gleam` file under `server/src/server/` and put this inside:

```gleam
pub type AppError {
  InternalServerError(String)
  Unauthorized
  BadRequest(String)
  NotFound(String)
  DatabaseError(pog.QueryError)
  HashError(argus.HashError)
  ValidationError(dict.Dict(String, String))
}
```

This way we can map all error into a custom AppError type.

Create `snippets.gleam` in `server/src/server/routes/` and put this inside:

```gleam
// server/src/server/routes/snippets.gleam

// method handler to route the request based on their method and path
pub fn snippets(ctx: context.Context, req: wisp.Request, id: String) {
  case req.method, id {
    http.Get, "" -> list_snippets(ctx, req)
    http.Get, id -> view_snippet(ctx, req, id)

    http.Post, _ -> {
      create_snippet(ctx, req)
    }
    http.Patch, id -> {
      update_snippet(ctx, req, id)
    }
    http.Delete, id -> {
      delete_snippet(ctx, req, id)
    }
    _, _ -> // other method will return method not allowed HTTP error
      wisp.method_not_allowed([http.Get, http.Post, http.Patch, http.Delete])
  }
}

// create a new CreateSnippet type
type CreateSnippet {
  CreateSnippet(title: String, content: String, ttl: Int)
}

// create a new JSON decoder for CreateSnippet
fn create_snippet_decoder() -> decode.Decoder(CreateSnippet) {
  use title <- decode.field("title", decode.string)
  use content <- decode.field("content", decode.string)
  use ttl <- decode.field("ttl", decode.int)
  decode.success(CreateSnippet(title:, content:, ttl:))
}

// handler to create snippet
fn create_snippet(ctx: context.Context, req: wisp.Request) {
  use json <- wisp.require_json(req) // we take JSON from request

  let result = {
    // decode the json
    use input <- result.try(
      decode.run(json, create_snippet_decoder())
      |> result.replace_error(errors.BadRequest(
        "missing title, content, or ttl",
      )),
    )

    // create a new expiry timestamp by getting current time and adding expiry duration
    let expires_at =
      timestamp.add(timestamp.system_time(), duration.hours(input.ttl))

    // save the snippet to DB
    snippets.create_snippet(ctx, input.title, input.content, expires_at)
  }

  // return a success message on Ok, otherwise handle the error
  case result {
    Ok(_) -> helpers.message_response("snippet created", 201)
    Error(err) -> errors.handle_error(req, err)
  }
}
```

You might ask why we create a new CreateSnippet type. We do this because we can't use the shared Snippet type because we only need a subset of the argument. We won't be receiving id and author\_id from the JSON and trying to decode a null would cause an error.

Create `helpers.gleam` in `server/src/server/` and put this inside:

```gleam
// get current time and turn it into UNIX seconds
pub fn current_time() -> Int {
  let #(now, _) =
    timestamp.system_time() |> timestamp.to_unix_seconds_and_nanoseconds()
  now
}

// create a simple JSON message response
pub fn message_response(message, status) {
  json.object([#("message", json.string(message))])
  |> json.to_string
  |> wisp.json_response(status)
}

// create a JSON response
pub fn json_response(data, item, status) {
  json.object([#(item, data)])
  |> json.to_string
  |> wisp.json_response(status)
}

// create an error JSON response
pub fn error_response(message, status) {
  json.object([#("error", json.string(message))])
  |> json.to_string
  |> wisp.json_response(status)
}
```

Add this to `errors.gleam` :

```gleam
pub type AppError {
  InternalServerError(String)
  Unauthorized
  BadRequest(String)
  NotFound(String)
  DatabaseError(pog.QueryError)
  HashError(argus.HashError)
  ValidationError(dict.Dict(String, String))
}

// handle AppError
pub fn handle_error(req: wisp.Request, err: AppError) -> wisp.Response {
  let message = app_error_to_string(err)

  // for each error case, log the error and send an error response
  case err {
    NotFound(_) -> {
      format_log(req, message) |> wisp.log_warning()
      helpers.error_response(message, 404)
    }
    Unauthorized -> {
      format_log(req, message) |> wisp.log_warning()
      helpers.error_response(message, 401)
    }
    InternalServerError(_) | DatabaseError(_) | HashError(_) -> {
      format_log(req, message) |> wisp.log_error()
      helpers.error_response("internal server error", 500)
    }
    BadRequest(_) -> {
      format_log(req, message) |> wisp.log_warning()
      helpers.error_response(message, 400)
    }
    ValidationError(_) -> {
      format_log(req, message) |> wisp.log_warning()
      helpers.error_response(message, 400)
    }
  }
}

// turn AppError into string
pub fn app_error_to_string(err: AppError) {
  case err {
    InternalServerError(err) -> err
    Unauthorized -> "unauthorized"
    BadRequest(err) -> err
    NotFound(val) -> val <> " not found"
    DatabaseError(err) -> pog_error_to_string(err)
    HashError(err) -> argus_error_to_string(err)
    ValidationError(err) ->
      "validation error: "
      <> {
        err
        |> dict.to_list
        |> list.map(fn(pair) { pair.0 <> ": " <> pair.1 })
        |> string.join(", ")
      }
  }
}

// turn Pog QueryError into string
fn pog_error_to_string(err: pog.QueryError) {
  case err {
    pog.ConstraintViolated(message:, constraint:, detail:) ->
      "constraint violated: { message:"
      <> message
      <> " constraint: "
      <> constraint
      <> " detail: "
      <> detail
      <> " }"
    pog.PostgresqlError(code:, name:, message:) ->
      "postgresql error: { code: "
      <> code
      <> " name: "
      <> name
      <> " message: "
      <> message
      <> " }"
    pog.UnexpectedArgumentCount(expected:, got:) ->
      "unexpected argument count: { expected: "
      <> int.to_string(expected)
      <> " got: "
      <> int.to_string(got)
      <> " }"
    pog.UnexpectedArgumentType(expected:, got:) ->
      "unexpected argument type: { expected: "
      <> expected
      <> " got: "
      <> got
      <> " }"
    pog.UnexpectedResultType(_) -> "unexpected result type"
    pog.QueryTimeout -> "query timeout"
    pog.ConnectionUnavailable -> "connection unavailable"
  }
}

// turn ArgusError into string (Argus is a password hashing libary, we'll get to that later)
fn argus_error_to_string(err: argus.HashError) {
  case err {
    argus.OutputPointerIsNull -> "output pointer is null"
    argus.OutputTooShort -> "output too short"
    argus.OutputTooLong -> "output too long"
    argus.PasswordTooShort -> "password too short"
    argus.PasswordTooLong -> "password too long"
    argus.SaltTooShort -> "salt too short"
    argus.SaltTooLong -> "salt too long"
    argus.AssociatedDataTooShort -> "associated data too short"
    argus.AssociatedDataTooLong -> "associated data too long"
    argus.SecretTooShort -> "secret too short"
    argus.SecretTooLong -> "secret too long"
    argus.TimeCostTooSmall -> "time cost too small"
    argus.TimeCostTooLarge -> "time cost too large"
    argus.MemoryCostTooSmall -> "memory cost too small"
    argus.MemoryCostTooLarge -> "memory cost too large"
    argus.TooFewLanes -> "too few lanes"
    argus.TooManyLanes -> "too many lanes"
    argus.PasswordPointerMismatch -> "password pointer mismatch"
    argus.SaltPointerMismatch -> "salt pointer mismatch"
    argus.SecretPointerMismatch -> "secret pointer mismatch"
    argus.AssociatedDataPointerMismatch -> "associated data pointer mismatch"
    argus.MemoryAllocationError -> "memory allocation error"
    argus.FreeMemoryCallbackNull -> "free memory callback null"
    argus.AllocateMemoryCallbackNull -> "allocate memory callback null"
    argus.IncorrectParameter -> "incorrect parameter"
    argus.IncorrectType -> "incorrect type"
    argus.InvalidAlgorithm -> "invalid algorithm"
    argus.OutputPointerMismatch -> "output pointer mismatch"
    argus.TooFewThreads -> "too few threads"
    argus.TooManyThreads -> "too many threads"
    argus.NotEnoughMemory -> "not enough memory"
    argus.EncodingFailed -> "encoding failed"
    argus.DecodingFailed -> "decoding failed"
    argus.ThreadFailure -> "thread failure"
    argus.DecodingLengthFailure -> "decoding length failure"
    argus.VerificationFailure -> "verification failure"
    argus.UnknownErrorCode -> "unknown error code"
  }
}

```

We're going to turn most error types into string to make it easy to log and to send error response to client. We send most error to the client except some internal error because it might contain sensitive information that can compromise server's security.

### List Snippets
