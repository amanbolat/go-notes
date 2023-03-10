# Go Notes

This project consists of different notes on Go, that I collected during all those years I've been using the language.
Some information might be copied from other sources, but I'll try to leave credits where it's possible.

## Rules of thumb

### 1. Use `context.Context`

Use `context.Context` in any method that does any IO operations or calls other methods that receive `context.Context` as a parameter.
IO operations might take long time, therefore we want to set timeouts and cancel them when needed. The same logic is 
applicable for nested methods that receive `context.Context` as a parameter.

### 2. Use custom `http.Client` and configure it appropriately

`http.Client` has defaults for timeouts and deadlines that might lead to leak of resources or bad performance in 
some cases. Rule of thumb is to NOT use default `http.Client` and create it when needed, configuring it for your use 
cases.

References:
- [The complete guide to Go net/http timeouts](https://blog.cloudflare.com/the-complete-guide-to-golang-net-http-timeouts/) – blog post from Cloudflare.

### 3. Use constants instead of literals

Package `net/http` provides public constants for HTTP status codes such as `http.StatusOK` or 
`http.StatusBadRequest`. Use those instead of `200` or `400` because it is easier to remember and makes your code 
more consistent. You should also create constants in your own code, even if the values is used only once, because in 
the future other people will work on your code and refactoring will be much easier to do. 

### 4. Never call `Fatal` anywhere, but from `main.go`

`Fatal` is not panic, it will halt your application, and it's not recoverable.

### 5. Try to not use panics

Don't use panics, return errors. Some people like to use panics and try to catch them as they are using the paradigm 
of `try and catch`. No, we don't do that in Go. There are some exceptions to that rule, but they are justifiable: 

- [time.Date](https://github.com/golang/go/blob/26fd1fba12a1a646c9f949e5e0a4fcb74e81c67e/src/time/time.go#L1469)
- [sql.Register](https://github.com/golang/go/blob/26fd1fba12a1a646c9f949e5e0a4fcb74e81c67e/src/database/sql/sql.go#L48)

### 6. Don't call `os.GetEnv` anywhere, but in `main.go`

`os.GetEnv` is mostly used to get some configuration for your application, and it is mostly done in `main.go`. 
Calling `os.GetEnv` you are probably implicitly getting the configuration when the application was already 
initialized, which might lead to bugs. Again, there are might be exceptions to this rule, but think twice before 
doing so.

### 7. Centralize your error logging

If you are developing a web app or any application that receives a request and sends a response back, you probably 
have a some kind of HTTP Router. Thus, you can handle logging of your errors in a single place, a middleware that 
intercepts all the responses, catch errors there and log them. No need to cal `log.Error` on every line before 
`return err` in your application. It's redundant, and you can easily forget to log the error somewhere in the future.

[Echo](https://github.com/labstack/echo) provides a really nice way to handle errors and log them in one place – 
[HTTPErrorHandler](https://github.com/labstack/echo/blob/24a30611dfc07e427dc771a16ef9bb0dd94c4c2e/echo.go#L129:2).

### 8. Use `defer` to do something in the end of a function

If there is any action that has to be done at the end of the function, use `defer`, otherwise you or someone else is 
going to append some code and the logic will be broken.

### 9. Don't use `fmt.Sprintf` for constructing SQL queries

It leads to SQL injections.

### 10. Always check slice's length before accessing it

It is really easy to access an element of a slice by index and cause panic, so always check the length.

<br/>

## Testing

### Database layer testing

Every application has Store/Database layer which mostly consists of CRUD operations. There are some tips on how to 
write those test and what to test.

**Quick tips:**

- Don't mock your database. Use [dockertest](https://github.com/ory/dockertest).
- Test your migrations.
- Run your CRUD operations more than once, run 1000 times.
- Use fake and random data. Library such as [gaofakeit](https://github.com/brianvoe/gofakeit) helps you with that.

Let's discuss every point mentioned above.

**Mocks**

You are testing your database layer, which means you are testing your implementation and not some abstract layer. 
Mocks are far away from a real Postgres or MySQL database. Testing against real database helps you to reveal some 
bugs before deploying your application to your real environment. Creating a new docker image for testing takes a few 
seconds, so don't be afraid of it.


**Migrations**

If your migrations are shipped with an application, test the migrations by calling `up` and `down` twice. It will 
help you avoid silly mistakes.

**1000 times**

During the tests create/read/update/delete more than once to catch deadlocks, avoid leaky resources and find some
hidden bugs. For example, the method that should return only one record, returns more than one, or deleting one
record in reality deletes more than one.

**Fake data**

Imagine that you mistakenly used `int8` in your database, but you are trying to save `int32` instead. If you use a 
constant value during the tests, you probably won't find the issue. Therefore, using some random data might help 
you to discover those problems during the tests.

<br/>

### Event-driven applications testing

You definitely have a part of some application that listens for some events incoming from a message broker and 
writes them to the database. Our tests should cover a few things:

- All the valid events were received successfully.
- All the data from events was mapped correctly and saved to DB.
- Invalid data was rejected and moved somewhere else.

**Counter**

Don't use `time.Sleep` and wait for a constant time until all the events were digested. On slow machines it might 
take longer and the test will fail. On faster machines you are wasting time. I suggest to have a custom counter 
injected into your `Subscriber/Listener` object, and read it every few seconds until all the event were drained. 
After that fetch all the saved records from the database and check for equality with sent events. This strategy also 
helps to count the number of invalid events.

**Containers**

Use `dockertest` or similar library to spin up the containers programmatically during the tests. Don't try to mock 
`kafka` or any other message broker. Creating `kafka` container for tests might be very slow and consume a lot of 
CPU and RAM, so I would suggest to use something which is Kafka API compatible, e.g. [redpanda](https://github.com/redpanda-data/redpanda/).
The latter doesn't need zookeeper, and it starts much faster than kafka. 
