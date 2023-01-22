# Go Notes

This project consists of different notes on Go, that I collected during all those years I've been using the language.
Some information might be copied from other sources, but I'll try to leave credits where it's possible.

## Quick notes

1. Use `context.Context` in any method as the first parameter that does any IO operation, because we want to control timeouts.
2. Configure `http.Client` with appropriate timeouts.
3. Always use methods `WithContext` when using 3rd party libraries set timeouts and cancel the context when necessary.
4. Use public constants such as `http.StatusBadRequest` for http status codes.
5. Never call `log.Fatal` or any other method that leads to `os.Exit` in any part of your application, except the main entry point, which is usually is `main.go`. Fatal is not panic, you can't recover from it.
6. Don't use panics, but return errors. Exceptions to this rule are rare, be mindful.
7. Never call `os.GetEnv` anywhere, but in `main.go`, or the package for application configuration.
8. Centralize your error logging in a single point. For example, `echo framework` provides a [HTTPErrorHandler](https://github.com/labstack/echo/blob/24a30611dfc07e427dc771a16ef9bb0dd94c4c2e/echo.go#L129:2).
9. Use `defer` for calling any method that in any case should be called at the end.
10. Never use `fmt.Sprintf` for constructing SQL queries! It will lead to SQL injections.
11. Always check slice's length before access the elements by index. 
