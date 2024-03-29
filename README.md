# Spinner
A simple language for creating web applications inspired by the simplicity of Sinatra. It then takes that simplicity and attempt to make it even simpler. No fluff, no cruft, just your app.

The route is made up of the following parts.
+ The methods
+ The path
+ The headers
+ The return status
+ The return value

Everything there is optional except the return value. The return value can be plain text (surrounded by quotes), json array, json object, or html/xml. It could also be a value returned from a function defined with the `fn` keyword.

## Default Values
+ When methods is absent, it defaults to `get, post`
+ When routes is absent, it defaults to `/`
+ when status is absent, it defaults to `200 OK`
+ A `Content-Type` header is automatically added and its value depends on the return value.

When the return value is a json object or array, Content-Type is `application/json`. When it's html, the Content-Type is `text/html`, and `text/plain` when it's plain text. It should be noted that Spinner listens on Port 8000 by default. So the following is a really short - and complete - spinner application:
```
{ message: "Hello, Avatar" }
```
In the above above example, because
1. The methods list is absent, it defaults to `get, post`. That's the `get` method and the `post` method.
2. The route is absent, it defaults to `/`.
3. The return status is absent, it defaults to 200 OK.

Also, because a json object is returned, the `Content-Type` header is set to `application/json`.

The following is an example which returns an html on a get request for /lake_laogai with a 403 (forbidden) status
```
get /lake_laogai 403
<html>
  <head>
    <title>Forbidden</title>
   </head>
   <body><p>Long Feng forbids you from entering here</p></body>
</html>
```
Because it returns an html, the  `Content-Type` header is set to `text/html`.  
The following will do the same thing except some information is logged and the return is done from a function. Note the use of the `fn` keyword used to define the function.
```
get /lake_laogai 403 fn{
  logger.info "Attempt to infilterate the Dai Lee"
  return
    <html>
      <head>
        <title>Forbidden</title>
      </head>
      <body><p>Long Feng forbids you from entering here</p></body>
    </html>
}
```
Specify the keyword `nil` to indicate an empty response body. When this is the case, the `Content-Type` header will be absent in the response and the `Content-Length` header would be set to 0. For example:
```
get, post /azula-love-interest 204 nil
```
In the above example, the response body for a get or post request to the route `/azula-love-interest` would be empty, the status would be set to 204, the `Content-Type` header would be absent and the `Content-Length` header would be set to 0.
