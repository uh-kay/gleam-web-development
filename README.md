# Introduction

In this book we're going to build an application called [Pastem](https://github.com/uh-kay/pastem), a JSON API for online clipboard service. What we're making is similar to [Pastebin](https://pastebin.com/).

Our API will support the following endpoints and action:

| Method | URL Pattern       | Description                       |
| ------ | ----------------- | --------------------------------- |
| GET    | /api/health       | Show application health           |
| POST   | /api/register     | Register a new user               |
| POST   | /api/tokens       | Create a new authentication token |
| POST   | /api/snippets     | Create a new snippet              |
| GET    | /api/snippets     | List snippets                     |
| GET    | /api/snippets/:id | View a snippet                    |
| PATCH  | /api/snippets/:id | Update a snippet                  |
| DELETE | /api/snippets/:id | Delete a snippet                  |

For database, we're going to use PostgreSQL.

### Prerequisite

#### Background Knowledge

While I try my best to explain everything, it might be faster for you to understand the book if you have basic understanding of Gleam's syntax. Here are a list of sources for you to learn:

* [Gleam Tour](https://tour.gleam.run)
* [Exercism's Gleam Track](https://exercism.org/tracks/gleam)

#### Gleam 1.14

The information in this book is correct as of Gleam version 1.14. If you need help installing Gleam, refer to the [official documentation](https://gleam.run/getting-started/installing).

#### Other Software

* curl (it should already be installed if you use Linux or MacOS)
* docker and docker compose (optional)<br>

### Convention

#### Code Block

```gleam
// path/to/file.gleam

pub fn main() -> Nil {
  io.println("Hello from server!")
}
```

Every code block looks like that and will have a path to the file.

#### Shell Command

`$ gleam run`

Shell command will look like that.

#### Further Reading

At the end of every chapter there might be a further reading section. This is completely optional but might help you with your projects.
