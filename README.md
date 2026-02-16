# Error Handling

### Case Expressions

In Gleam, function that can fail returns Result type. Result type evaluates into Ok and Error case. Let's look at an example:

```gleam
pub type ParseError {
  NotNumber
  NotAllowed(num: String )
}

// Turns "1" and "2" into Int, otherwise return error.
pub fn parse_int(str: String) -> Result(Int, ParseError) {
  case str {
    "1" -> Ok(1)
    "2" -> Ok(2)
    "3" -> Error(NotAllowed("3"))
    _ -> Error(NotNumber)
  }
}
```

parse\_int() take a string argument and turns it into Int if the argument is "1" or "2", otherwise return ParseError. (I admit the function name is misleading, but let's not worry about that :D) Now let's see how we use the function and handle the error:

```gleam
pub fn main() {
  echo parse_int("1") // Ok(1)
  echo parse_int("3") // Error(NotAllowed("3"))
  echo parse_int("hello") // Error(NotNumber)
  
  case parse_int("world") {
    Ok(num) -> echo num // this won't run
    Error(err) -> echo err // this will run
  }
}
```

### result.try()

In result standard library module, we have various function to handle error. One of them is result.try().

```gleam
result.try(parse_int("1"), fn(num) {
  Ok(num + 1)
})
  
// parse_int("1") evaluates to Ok(1)
result.try(Ok(1), fn(num) {
  Ok(num + 1)
})
```

result.try() is a function that takes in a Result and a function uses the value of the Result. In the example, we use `parse_int("1")` as the first argument and an anonymous function that takes in the Ok value of `parse_int("1")` and add one.

result.try() returns a Result. So to use it, we'd do something like this:

```gleam
pub fn main() {
  let result = result.try(parse_int("1"), fn(num) {
    Ok(num + 1)
  })
  
  case result {
    Ok(num) -> echo num // 2
    Error(err) -> echo err
  }
}
```
