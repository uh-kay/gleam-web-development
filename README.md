# Error Handling

In Gleam, function that can fail returns Result type. Result type evaluates into Ok and Error case. Let's look at an example:

```gleam
pub type ParseError {
  NotNumber
}

// Turns "1" and "2" into Int, otherwise return error.
pub fn parse(str: String) -> Result(Int, String) {
  "1" -> Ok(1)
  "2" -> Ok(2)
  _ -> Error("cannot parse string into int")
}
```

parse() take a string argument and turns it into Int if the argument is "1" or "2", otherwise it will return a String error. Now let's see how we use the function and handle the error:

```gleam
pub fn main() {
  echo parse("1") // Ok(1)
  echo parse("hello") // Error("cannot parse string into int")
  
  case parse("world") {
    Ok(num) -> echo num // 1
    Error(err) -> echo err // "cannot parse string into int"
  }
}
```
