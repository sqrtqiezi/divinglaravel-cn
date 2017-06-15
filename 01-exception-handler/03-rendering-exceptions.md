# Rendering Exceptions

While rendering an exception Laravel checks if the exception class has a `render()` method, if so it just uses the output of this method to build the response, you can return anything from this method as you normally do within a controller method.

Laravel tries to convert exceptions into a displayable format depending on the expected response format whether it's HTML or JSON, it first converts various exceptions formats to a simple HTTPException:

```php
if ($e instanceof ModelNotFoundException) {
    $e = new NotFoundHttpException($e->getMessage(), $e);
} elseif ($e instanceof AuthorizationException) {
    $e = new HttpException(403, $e->getMessage());
} elseif ($e instanceof TokenMismatchException) {
    $e = new HttpException(419, $e->getMessage());
}
```

And then it handles some exceptions in a special way:

`Illuminate\Http\Exceptions\HttpResponseException` is a special exception in that it already contains the response, so Laravel just returns the response from that exception.

## Rendering Authentication Exceptions

The `Illuminate\Auth\AuthenticationException` is handled using a method in a `unauthenticated()` method inside your `App\Exceptions`, by default it redirects the user to a `/login` URL in case the expected response format is HTML or returns a JSON object with a 401 status code:

```json
{"message" : "Unauthenticated."}
```

Of course you can change the behaviour in this method as you want.

## Rendering Validation Exceptions

In case of a `Illuminate\Validation\ValidationException`, laravel redirects the user to the previous URI with the input given in the request as well as an error bag, this gives you the ability to check if the $error variable contains any `$errors` that you can display:

```php
@if (count($errors) > 0)
    <div class="alert alert-danger">
        <ul>
            @foreach ($errors->all() as $error)
                <li>{{ $error }}</li>
            @endforeach
        </ul>
    </div>
@endif
```

In case the expected response format is JSON Laravel returns a JSON object with a 422 status code, it looks like this:

```json
{
  "message": "The given data failed to pass validation.",
  "errors": {
    "name": [
        "The name field is required.",
        "The name field must be a string."
    ]
  }
}
```

## Rendering other exceptions

At this point all the special exceptions are either converted to a simple HTTPException or rendered using a special way, as for the rest of exceptions laravel first checks if the expected response format is HTML or JSON and then acts accordingly:

### How does laravel detects the expected response format?

Laravel uses the `expectsJson()` method inside `Illuminate\Http\Request`, this methods checks if a `X-Requested-With` header is present and holds a `XMLHttpRequest` value, this header is set by most JavaScript frameworks and laravel uses it to assume that the request is an Ajax request, however Laravel also checks for a `X-PJAX` header on the request and just returns false if it's present since that indicates that the response should not be in JSON format but rather a regular HTML response.

Finally it looks into the `Accept` header and see if the content of the header implies that it expects JSON in the response.

### So if the response expects JSON how does laravel convert the exception to JSON?

In case the `app.debug` configuration is set to true laravel converts the exception to a JSON format with the following structure:

```json
{
    "message": "...",
    "file": "...",
    "line": 0,
    "trace": "..."
}
```

This helps a lot while working in a development environment in that it gives the developer insights about what went wrong while the request was being handled, but of course the `app.debug` options shouldn't be set to true in a production server since it may expose sensitive information, in that case laravel checks if the exception is an HTTPException and returns the exception message a JSON structure:

```json
{
    "message": "..."
}
```

However, if the exception is not an HTTP Exception, 500, Laravel just responds with a "Server Error" message:

```json
{
    "message": "Server Error"
}
```

> If you're fine with displaying the exception message to people consuming your API then throw an HTTP Exception, otherwise laravel will protect your data by hiding the actual exception message and only show "Server Error".

### What happens when an HTML response is expected?

First laravel checks if you have any views inside your `resources/views/errors` directory with the name of the response status code, for example `500.blade.php`, if so Laravel renders that view and presents it to the end user.

If no view was found Laravel will use Symfony's exception handler to display a nice view with information about the exception in case `app.debug` is on, if it's not then a "Whoops, looks like something went wrong." will appear to the end user.
