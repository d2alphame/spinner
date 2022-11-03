# Spinner
A simple language for creating web applications. It is inspired by the simplicity of Sinatra. It takes that simplicity and attempts to make it even simpler :D

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

When the return value is a json object or array, Content-Type is `application/json`. When it's html, the Content-Type is `text/html`, and `text/pain` when it's plain text. So the following is a really short - and complete - spinner application:
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
