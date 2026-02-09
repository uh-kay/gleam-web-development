# Getting Started

### Project Setup

Let's start by creating `pastem` directory. I'm going to create it on `$HOME/Projects/pastem` but you're free to create it anywhere.

`$ mkdir pastem && cd pastem`

#### Generating the project

Inside this folder we are going to initialized a new project.

`$ gleam new server && cd server`

Inside the server directory, you'll see this:

```
.
├── gleam.toml
├── README.md
├── src
│   └── server.gleam
└── test
    └── server_test.gleam
```

* `gleam.toml` contains configuration for the project.
* `src/` is where all of our code will be.
* `test/` is where all of our test code will be.

#### Hello, world!

Open `src/server.gleam`. Inside you'll see this:

```gleam
// server/src/server.gleam
import gleam/io

pub fn main() -> Nil {
  io.println("Hello from server!")
}
```

Try running `gleam run`. You should see `Hello from server` printed on the terminal. Congratulations, you're now a [Gleamlins](https://gleam.run/frequently-asked-questions/#What-are-Gleam-programmers-called)!

### Basic HTTP Server

Before writing our first code, let's start by installing a couple of library.

`$ gleam add mist wisp wisp_mist gleam_erlang gleam_time`&#x20;

This will install [mist](https://github.com/rawhat/mist) web server and [wisp](https://github.com/gleam-wisp/wisp) framework.

Inside `server/src/` run `mkdir -p server/routes`. Inside `server/src/server/` run `touch errors.gleam middleware.gleam router.gleam routes/health.gleam`. Our project structure now look like this:

```
.
├── server
│   ├── errors.gleam
│   ├── middleware.gleam
│   ├── router.gleam
│   └── routes
│       └── health.gleam
└── server.gleam
```

Inside `server.gleam`, delete all the code inside main function and replace it with this:

```gleam
// server/src/server.gleam
pub fn main() {
  // This sets the logger to print INFO level logs, and other sensible defaults for a 
  // web application.
  wisp.configure_logger()

  // Here we generate a secret key, but later we'll load it from environment variables.
  let secret_key_base = wisp.random_string(64)
  
  // Create a new request handler to process all the request.
  let handler = handle_request(_)

  // Start the Mist web server on port 8000.
  let assert Ok(_) = 
    handler
    |> wisp_mist.handler(secret_key_base)
    |> mist.new
    |> mist.port(8000)
    |> mist.start
  
  // The web server runs in new Erlang process, so we will put it to sleep while it 
  // works concurrently.
  process.sleep_forever()
}
```

Inside router.gleam, write this:

```gleam
// server/src/server/router.gleam
pub fn handle_request(req: wisp.Request) -> wisp.Response {
  // Pass the request to the global middleware.
  use req <- middleware(req)
    
  // path_segments() returns a list of string that we can pattern match to handle route. 
  // The code below will match /api/health and call health handler.
  case wisp.path_segments(req) {
    ["api", "health"] -> health()
  }
}
```

Inside middleware.gleam, write this:

```gleam
// server/src/server/middleware.gleam
pub fn middleware(
  req: wisp.Request,
  handle_request: fn(wisp.Request) -> wisp.Response,
) -> wisp.Response {
  // Take a the request and override the method if _method query parameter exists.
  let req = wisp.method_override(req)
  // Logs all the request.
  use <- log_request(req)
  // Rescue all crashes and return 500 HTTP status code.
  use <- wisp.rescue_crashes
  // Convert HEAD request to GET, handle the request, then discard the body.
  use req <- wisp.handle_head(req)
  // Protects against Cross-Site Request Forgery (CSRF) attacks by checking the host 
  // request header against the origin header or referer header.
  use req <- wisp.csrf_known_header_protection(req)
  
  // Handle the request
  handle_request(req)
}

pub fn log_request(
  req: wisp.Request,
  handler: fn() -> wisp.Response,
) -> wisp.Response {
  errors.format_log(req, "")
  |> wisp.log_info
  handler()
}
```

Inside routes/health.gleam, write this:

```gleam
// server/src/server/routes/health.gleam
pub fn health() -> Response {
  // Create a new json object, pass in a list of tuple (string key and json value) 
  // as its argument.
  json.object([#("status", json.string("available"))])
  |> json.to_string // turn the json object into string
  |> wisp.json_response(200) 
  // pass the string as response body and set 200 as the HTTP status code
}
```

Inside errors.gleam, write this:

```gleam
// server/src/server/errors.gleam
pub fn format_log(req: wisp.Request, message: String) {
  timestamp.to_rfc3339(timestamp.system_time(), calendar.local_offset())
  <> " "
  <> http_method_to_string(req.method)
  <> " "
  <> req.path
  <> " "
  <> message
}

fn http_method_to_string(method: http.Method) {
  case method {
    http.Get -> "GET"
    http.Post -> "POST"
    http.Head -> "HEAD"
    http.Put -> "PUT"
    http.Delete -> "DELETE"
    http.Trace -> "TRACE"
    http.Connect -> "CONNECT"
    http.Options -> "OPTIONS"
    http.Patch -> "PATCH"
    http.Other(_) -> "OTHER"
  }
}
```

Let's take a step back and review what we wrote. The `main` function creates a new web server running on port 8000. To handle all the request coming in, we wrote `handle_request` function.

Inside handle\_request, we use the `use` syntax to wrap our logic in `middleware`. Middleware is like a series of layers that the request must pass through (logging, crash recovery, and security checks) before reaching our actual route logic.

Finally, we use pattern matching on `wisp.path_segments(req)` to route the request. If a user hits `/api/health`, the `health` function is called, returning a JSON object confirming the server is "available" with a 200 OK status.

To start the server, run `gleam run`. Now, if we visit `http://localhost:8000/api/health`, we'll see:

```bash
$ curl -i http://localhost:8000/api/health
HTTP/1.1 200 OK
content-type: application/json; charset=utf-8
content-length: 22
date: Thu, 05 Feb 2026 17:16:07 GMT
connection: keep-alive
```

### Loading Environment Variables

Inside our main function we hardcoded some value, let's change those. We want this to be loaded from `.env` file. We can install another library to load this for use, but I usually use [direnv](https://github.com/direnv/direnv). Check the git repository for the install instruction.

After you install direnv, put this line inside your .bashrc: `eval "$(direnv hook bash)"`. Then inside `server` directory, create `.env` file and write this:

```bash
PORT=8000
SECRET_KEY_BASE=super_secret_string
```

Run `direnv allow` to load the env variables. Then run `gleam add envoy`. This library allows us to use the value of environment variables instead of hardcoding it.

Add this to `server.gleam`:

```gleam
// server/src/server.gleam
pub fn main() {
  wisp.configure_logger()

  let secret_key_base =
    result.unwrap(envoy.get("SECRET_KEY_BASE"), "changethis") // get secret key from .env or use the default value "changethis"
  let port_str = result.unwrap(envoy.get("PORT"), "8000") // get port from .env or use the default value 8000
  let assert Ok(port) =  int.parse(port_str)
  
  let handler = handle_request(_)

  let assert Ok(_) = 
    handler
    |> wisp_mist.handler(secret_key_base)
    |> mist.new
    |> mist.port(port)
    |> mist.start
  
  process.sleep_forever()
}
```

Note: Don't forget to put `.env` inside `.gitignore`.

### Further Reading

* Wisp examples: [https://github.com/gleam-wisp/wisp/tree/main/examples](https://github.com/gleam-wisp/wisp/tree/main/examples)
