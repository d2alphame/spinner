# Spinner
A simple language for creating web applications much like sinatra

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

