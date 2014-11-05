phly/http
=========

[![Scrutinizer Code Quality](https://scrutinizer-ci.com/g/phly/http/badges/quality-score.png?b=master)](https://scrutinizer-ci.com/g/phly/http/?branch=master)
[![Code Coverage](https://scrutinizer-ci.com/g/phly/http/badges/coverage.png?b=master)](https://scrutinizer-ci.com/g/phly/http/?branch=master)
[![Scrutinizer Build Status](https://scrutinizer-ci.com/g/phly/http/badges/build.png?b=master)](https://scrutinizer-ci.com/g/phly/http/build-status/master)

`phly/http` is a PHP package containing implementations of the [proposed PSR HTTP message interfaces](https://github.com/php-fig/fig-standards/blob/master/proposed/http-message.md), as well as a "server" implementation similar to [node's http.Server](http://nodejs.org/api/http.html).

This package exists:

- to provide a proof-of-concept of the proposed PSR HTTP message interfaces with relation to server-side applications.
- to provide a node-like paradigm for PHP front controllers.
- to provide a common methodology for marshaling a request from the server environment.

Installation and Requirements
-----------------------------

Install this library using composer:

```console
$ composer require "psr/http-message:~0.5.1@dev" "phly/http:~1.0-dev@dev"
```

`phly/http` has the following dependencies (which are managed by Composer):

- `psr/http-message`, which defines interfaces for HTTP messages, including requests and responses. `phly/http` provides implementations of each of these.

Contributing
------------

- Please write unit tests for any features or bug reports you have.
- Please run unit tests before opening a pull request. You can do so using `./vendor/bin/phpunit`.
- Please run CodeSniffer before opening a pull request, and correct any issues. Use the following to run it: `./vendor/bin/phpcs --standard=PSR2 --ignore=test/Bootstrap.php src test`.

Usage
-----

Usage will differ based on whether you are writing an HTTP client, or a server-side application.

For HTTP client purposes, you will create and populate an `OutoingRequest` instance, and the client should return an `IncomingResponse` instance.

For server-side applications, you will create an `IncomingRequest` instance, and populate and return an `OutgoingResponse` instance.

### HTTP Clients

A client will _send_ a request, and _return_ a response. As a developer, you will _create_ and _populate_ the request, and then _introspect_ the response. Thus, requests are mutable, and responses are immutable; the request represents the arguments you are marshaling to send, and the response is the _product_ of computation.

```php
// Create a request
$request = new Phly\Http\OutgoingRequest();
$request->setUrl('http://example.com');
$request->setMethod('PATCH');
$request->addHeader('Authorization', 'Bearer ' . $token);
$request->addHeader('Content-Type', 'application/json');
$request->getBody()->write(json_encode($data));

$response = $client->send($request);

printf("Response status: %d (%s)\n", $response->getStatusCode(), $response->getReasonPhrase());
printf("Headers:\n");
foreach ($response->getHeaders() as $header => $values) {
  printf("    %s: %s\n", $header, implode(', ', $values));
}
printf("Message:\n%s\n", $response->getBody());
```

(Note: phly/http does NOT ship with a client implementation; the above is just an illustration of a possible implementation.)

### Server-Side Applications

Server-side applications will need to marshal the incoming request based on superglobals, and will then populate and send a response.

#### Marshaling an incoming request

PHP contains a plethora of information about the incoming request, and keeps that information in a variety of locations. `Phly\Http\IncomingRequestFactory::fromGlobals()` can simplify marshaling that information into a request instance.

You can call the factory method with or without the following arguments, in the following order:

- `$server`, typically `$_SERVER`
- `$query`, typically `$_GET`
- `$body`, typically `$_POST`
- `$cookies`, typically `$_COOKIE`
- `$files`, typically `$_FILES`

The method will then return a `Phly\Http\IncomingRequest` instance. If any argument is omitted, the associated superglobal will be used.

```php
$request = Phly\Http\IncomingRequestFactory::fromGlobals(
  $_SERVER,
  $_GET,
  $_POST,
  $_COOKIE,
  $_FILES
);
```

#### Manipulating the response

Use the response object to add headers and provide content for the response.

```php
// Write to the response body:
$response->getBody()->write("some content\n");

// Multiple calls to write() append:
$response->getBody()->write("more content\n"); // now "some content\nmore content\n"

// Add headers
// Note: headers do not need to be added before data is written to the body!
$response->setHeader('Content-Type', 'text/plain');
$response->addHeader('X-Show-Something', 'something');
```

#### "Serving" an application

`Phly\Http\Server` mimics a portion of the API of node's `http.Server` class. It invokes a callback, passing it an `IncomingRequest`, an `OutgoingResponse`, and optionally a callback to use for incomplete/unhandled requests.

You can create a server in one of three ways:

```php
// Direct instantiation, with a callback handler, request, and response
$server = new Phly\Http\Server(
    function ($request, $response, $done) {
        $response->getBody()->write("Hello world!");
    },
    $request,
    $response
);

// Using the createServer factory, providing it with the various superglobals:
$server = Phly\Http\Server::createServer(
    function ($request, $response, $done) {
        $response->getBody()->write("Hello world!");
    },
    $_SERVER,
    $_GET,
    $_POST,
    $_COOKIE,
    $_FILES
);

// Using the createServerFromRequest factory, and providing it a request:
$server = Phly\Http\Server::createServerfromRequest(
  function ($request, $response, $done) {
      $response->getBody()->write("Hello world!");
  },
  $request
);
```

Server callbacks can expect up to three arguments, in the following order:

- `$request` - the request object
- `$response` - the response object
- `$done` - an optional callback to call when complete

Once you have your server instance, you must instruct it to listen:

```php
$server->listen();
```

At this time, you can optionally provide a callback to `listen()`; this will be passed to the handler as the third argument (`$done`):

```php
$server->listen(function ($error = null) {
    if (! $error) {
        return;
    }
    // do something with the error...
});
```

Typically, the `listen` callback will be an error handler, and can expect to receive the error as its argument.

API
---

### OutgoingRequest Message

`Phly\Http\OutgoingRequest` implements `Psr\Http\Message\OutgoingRequestInterface`, and is intended for client-side requests. It includes the following methods:

```php
class OutgoingRequest
{
    public function __construct($stream = 'php://memory');
    public function setMethod($method);
    public function setUrl($url);
    public function setProtocolVersion($version);
    public function getMethod();
    public function getUrl();
    public function setHeader($header, $value);
    public function addHeader($header, $value);
    public function removeHeader($header);
    public function setBody(Psr\Http\Message\StreamableInterface $body);
    public function getProtocolVersion();
    public function getBody();
    public function getHeaders();
    public function hasHeader($header);
    public function getHeader($header);
    public function getHeaderAsArray($header);
}
```

### IncomingResponse Message

`Phly\Http\IncomingResponse` implements `Psr\Http\Message\IncomingResponseInterface`, and models the response from a client-side request. It includes the following methods:

```php
class IncomingResponse
{
    public function __construct($statusCode, array $headers, $stream, $reasonPhrase = NULL);
    public function getStatusCode();
    public function getReasonPhrase();
    public function getProtocolVersion();
    public function getBody();
    public function getHeaders();
    public function hasHeader($header);
    public function getHeader($header);
    public function getHeaderAsArray($header);
}
```

### IncomingRequest Message

For server-side applications, `Phly\Http\IncomingRequest` implements `Psr\Http\Message\IncomingRequestInterface`, which provides access to the elements of an HTTP request, as well as uniform access to the various elements of incoming data. The methods included are:

```php
class IncomingRequest
{
    public function __construct(
        $url,
        $method = NULL,
        array $headers = [],
        $stream = 'php://input',
        array $serverParams = [],
        array $cookieParams = [],
        array $queryParams = [],
        array $bodyParams = [],
        array $fileParams = [],
        array $attributes = [],
        $protocolVersion = NULL
    );
    public function getServerParams();
    public function getCookieParams();
    public function getQueryParams();
    public function getFileParams();
    public function getBodyParams();
    public function getAttributes();
    public function getAttribute($attribute, $default = NULL);
    public function setAttributes(array $values);
    public function setAttribute($attribute, $value);
    public function setBodyParams(array $values);
    public function getMethod();
    public function getUrl();
    public function getProtocolVersion();
    public function getBody();
    public function getHeaders();
    public function hasHeader($header);
    public function getHeader($header);
    public function getHeaderAsArray($header);
}
```

Most data MUST be injected during instantiation, and SHOULD NOT be changed during the lifetime of the object. The exception to this rule are _attributes_, which are considered any data derived from the request instance (e.g., decrypted cookies, deserialized non-form body content, user data identified from Authorization headers, etc.).

### OutgoingResponse Message

`Phly\Http\OutgoingResponse` provides an implementation of `Psr\Http\Message\OutgoingResponseInterface`, a mutable response to be used to aggregate response information for a server-side application, including headers and message body content. It includes the following:

```php
class OutgoingResponse
{
    public function __construct($stream = 'php://memory');
    public function setStatus($code, $reasonPhrase = NULL);
    public function setProtocolVersion($version);
    public function getStatusCode();
    public function getReasonPhrase();
    public function setHeader($header, $value);
    public function addHeader($header, $value);
    public function removeHeader($header);
    public function setBody(Psr\Http\Message\StreamableInterface $body);
    public function getProtocolVersion();
    public function getBody();
    public function getHeaders();
    public function hasHeader($header);
    public function getHeader($header);
    public function getHeaderAsArray($header);
}
```

#### IncomingRequestFactory

This static class can be used to marshal an `IncomingRequest` instance from the PHP environment. The primary entry point is `Phly\Http\IncomingRequestFactory::fromGlobals(array $server, array $query, array $body, array $cookies, array $files)`. This method will create a new `IncomingRequest` instance with the data provided. Examples of usage are:

```php
// Returns new IncomingRequest instance, using values from superglobals:
$request = IncomingRequestFactory::fromGlobals(); 

// or

// Returns new IncomingRequest instance, using values provided (in this
// case, equivalent to the previous!)
$request = RequestFactory::fromGlobals(
  $_SERVER,
  $_GET,
  $_POST,
  $_COOKIE,
  $_FILES
);
```

### URI

`Phly\Http\Uri` models and validates URIs. The request object casts URLs to `Uri` objects, and returns them from `getUrl()`, giving an OOP interface to the parts of a URI. It implements `__toString()`, allowing it to be represented as a string and `echo()`'d directly. The following methods are pertinent:

```php
class Uri
{
    public static function fromArray(array $parts);
    public function __construct($uri);
    public function isValid();
    public function setPath($path);
}
```

`fromArray()` expects an array of URI parts, and should contain 1 or more of the following keys:

- scheme
- host
- port
- path
- query
- fragment

`setPath()` accepts a path, but does not actually change the `Uri` instance; it instead returns a clone of the current instance with the new path.

The following properties are exposed for read-only access:

- scheme
- host
- port
- path
- query
- fragment

### Stream

`Phly\Http\Stream` is an implementation of `Psr\Http\Message\StreamableInterface`, and provides a number of facilities around manipulating the composed PHP stream resource. The constructor accepts a stream, which may be either:

- a stream identifier; e.g., `php://input`, a filename, etc.
- a PHP stream resource

If a stream identifier is provided, an optional second parameter may be provided, the file mode by which to `fopen` the stream.

Request objects by default use a `php://input` stream set to read-only; Response objects by default use a `php://memory` with a mode of `wb+`, allowing binary read/write access.

In most cases, you will not interact with the Stream object directly.

### Server

`Phly\Http\Server` represents a server capable of executing a callback. It has four methods:

```php
class Server
{
    public function __construct(
        callable $callback,
        Psr\Http\Message\IncomingRequestInterface $request,
        Psr\Http\Message\OutgoingResponseInterface $response
    );
    public static function createServer(
        callable $callback,
        array $server,  // usually $_SERVER
        array $query,   // usually $_GET
        array $body,    // usually $_POST
        array $cookies, // usually $_COOKIE
        array $files    // usually $_FILES
    );
    public static function createServerFromRequest(
        callable $callback,
        Psr\Http\Message\IncomingRequestInterface $request,
        Psr\Http\Message\OutgoingResponseInterface $response = null
    );
    public function listen(callable $finalHandler = null);
}
```

You can create an instance of the `Server` using any of the constructor, `createServer()`, or `createServerFromRequest()` methods. If you wish to use the default request and response implementations, `createServer($middleware, $_SERVER, $_GET, $_POST, $_COOKIE, $_FILES)` is the recommended option, as this method will also marshal the `IncomingRequest` object based on the PHP request environment.  If you wish to use your own implementations, pass them to the constructor or `createServerFromRequest()` method (the latter will create a default `OutgoingResponse` instance if you omit it).

`listen()` executes the callback. If a `$finalHandler` is provided, it will be passed as the third argument to the `$callback` registered with the server.
