# 渲染异常

在渲染异常的时候，Laravel 会检查异常类是否有 `render()` 方法。如果有，它就只是使用该方法的输出来构建响应。你可以像平常在控制器方法中做的那样从这个方法返回任何东西。

Laravel 尝试将异常转换为你事先决定好的可显示的响应格式。而无论是 HTML 或者 JSON，它都会先将各种各样的异常格式转换为简单的 HTTPException：

```php
if ($e instanceof ModelNotFoundException) {
    $e = new NotFoundHttpException($e->getMessage(), $e);
} elseif ($e instanceof AuthorizationException) {
    $e = new HttpException(403, $e->getMessage());
} elseif ($e instanceof TokenMismatchException) {
    $e = new HttpException(419, $e->getMessage());
}
```

然后它用一种特殊的方式处理一些异常：

`Illuminate\Http\Exceptions\HttpResponseException` 是一种特殊的异常，在它里面已经包含了响应，所以 Laravel 只是返回该异常的响应。

### 渲染认证异常

 使用 `App\Exceptions` 的 `unauthenticated()` 中的方法处理 `Illuminate\Auth\AuthenticationException` 。默认情况下，如果事先决定的响应格式是带着 401 状态码的 HTML 或者返回 JSON 对象，那程序就会将用户重定向到 `/login` URL 上。

```json
{"message" : "Unauthenticated."}
```

当然你也可以根据需要更改这个方法中的行为。

### 渲染验证异常

如果是 `Illuminate\Validation\ValidationException`，Laravel 将用户重定向到前一个 URI，并在给出请求中输入的内容以及错误信息，这样就可以检查 `$errors` 变量是否包含任何可以显示的错误：

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

如果事先决定的响应格式是 JSON，Laravel 就返回一个带有 422 状态码的 JSON 对象，它看起来像这样：

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

### 渲染其他异常

此时，所有特殊异常都被转换成一个简单的 HTTPException 或者用一种特殊的方法来渲染。至于其他的异常，Laravel 会先检查事先决定的响应格式是 HTML 还是 JSON，然后相应地采取行动：

#### Laravel 如何检测事先决定的响应格式？

Laravel 使用 `Illuminate\Http\Request` 中的 `expectsJson()` 方法，这个方法检查 header 是否存在 `X-Requested-With`，如果存在就保存  `XMLHttpRequest` 的值。这个 header 是由大多数 JavaScript 框架设置，并且 Laravel 使用它来假定请求是一个 Ajax 请求，还会根据请求检查 header `X-PJAX`，如果存在，则返回 false。因为这表示响应不应该是 JSON 格式，而是一个常规的 HTML 响应。

最后它会查看 header 的 `Accept`，看看 header 文件的内容是否意味着它需要在响应中得到 JSON。

#### 那么如果是希望响应 JSON 那又是如何将异常转换为 JSON 呢？

如果将 `app.debug` 设置为 true，Laravel 会将该异常转换为具有以下结构的 JSON 格式：

```json
{
    "message": "...",
    "file": "...",
    "line": 0,
    "trace": "..."
}
```

这对在开发环境中工作时有很大帮助，因为它为开发人员提供了处理请求时发生了什么问题的了解渠道。但是当然，在生产服务器中应该不应该将 `app.debug` 设置为 true，因为它可能会暴露敏感信息。在这种情况下，laravel 会检查异常是否是 HTTPException，并返回 JSON 结构发的异常消息：

```json
{
    "message": "..."
}
```

然而，如果异常不是 HTTP 异常，500，Laravel 只会响应「服务器错误」的消息：

```json
{
    "message": "Server Error"
}
```

> 如果设置了向使用 API 的用户显示异常消息，则抛出 HTTP 异常。否则 Laravel 将通过隐藏实际异常消息并仅显示「服务器错误」来保护你的数据。



#### 当事先约定的 HTML 响应时会发生什么？

首先 Laravel 会检查 `resources/views/errors` 目录中是否有包含响应状态代码名称的任何视图，例如 `500.blade.php`。如果有，Laravel 会渲染该视图并将其呈现给用户。

如果没有找到视图，Laravel 会使用 Symfony 的异常处理程序来显示一个友好的视图。而且如果 `app.debug` 被打开了，那这个视图还会包含有关异常的信息。但如果没有，最终出现在用户面前的会是「Whoops, looks like something went wrong.」这句话。