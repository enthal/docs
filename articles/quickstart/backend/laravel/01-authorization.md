---
title: Authorization
description: This tutorial demonstrates how to add authorization to a Laravel API.
topics:
    - quickstart
    - backend
    - laravel
github:
   path: 01-Authorization-RS256
contentType: tutorial
useCase: quickstart
---

This quickstart covers building an API protected by an Auth0-issued access token. This type of API is typically consumed by:

- Mobile, desktop, and other native applications using the [Native login flow](https://auth0.com/docs/flows/concepts/auth-code-pkce)
- CLIs, daemons, or services running on your back-end using the [Client Credentials Flow](https://auth0.com/docs/flows/concepts/client-credentials)

If this API is only consumed by a web application on the same domain (in the case of AJAX actions or lazy loading content for an authenticated user) then the API protection should be handled by the application itself and the [login flow secured by Auth0](https://auth0.com/docs/flows/concepts/auth-code).

<%= include('../_includes/_api_auth_preamble') %>

<%= include('../../../_includes/_api_auth_intro') %>

<%= include('../_includes/_api_create_new') %>

## Validate Access Tokens

### Install dependencies

Protecting your Laravel API requires a middleware which will check for and verify a bearer token in the `Authorization` header of an incoming HTTP request. We'll do that using tools provided by the [Auth0 Laravel](https://github.com/auth0/laravel-auth0) package.

::: note
**[Composer](https://getcomposer.org/)** is a tool for dependency management in PHP. It allows you to declare the dependent libraries your project needs and it will install them in your project for you. See Composer's [getting started](https://getcomposer.org/doc/00-intro.md) doc for information on how to use it.
:::

```bash
composer require auth0/login
```

### Configure the plugin

The **laravel-auth0** plugin comes with a configuration file that can be generated using [Artisan](https://laravel.com/docs/5.7/artisan). First, generate the configuration file from the command line:

```bash
php artisan vendor:publish --provider "Auth0\Login\LoginServiceProvider"
```

After the file is generated, it will be located at `config/laravel-auth0.php`. Edit this file to add the configuration values needed to verify incoming tokens:

```php
// config/laravel-auth0.php
return [
	// ...
	'authorized_issuers' => [ 'https://${account.namespace}/' ],
	// ...
	'api_identifier' => '${apiIdentifier}',
	// ...
	'supported_algs' => [ 'RS256' ],
	// ...
];
```

In more detail:

* `authorized_issuers` is an array of allowed token issuers. In this case, it would simply be an array with just your tenant URL.
* `api_identifier` is the **Identifier** field of the API [created above](#configure-auth0-apis).
* `supported_algs` is the **Signing Algorithm** field of the API [created above](#configure-auth0-apis). This value should be an array but only have a single value, `RS256`.

### Configure Apache

By default, Apache does not parse `Authorization` headers from incoming HTTP requests. You may need to add the following to the `.htaccess` file for your application:

```bash
RewriteEngine On
RewriteCond %{HTTP:Authorization} ^(.*)
RewriteRule .* - [e=HTTP_AUTHORIZATION:%1]
```

## Protect API Endpoints

<%= include('../_includes/_api_endpoints') %>

For the two `private` API routes, we'll need a middleware to check for a bearer token in an `Authorization` header for the request and then verify that the token is valid. We'll create that middleware using the `make:middleware` Artisan command:

```bash
php artisan make:middleware CheckJWT
```

Now, let's implement the `handle()` method that Laravel will call automatically for the route:

```php
<?php
// app/Http/Middleware/CheckJWT.php

namespace App\Http\Middleware;

use Auth0\Login\Contract\Auth0UserRepository;
use Auth0\SDK\Exception\CoreException;
use Auth0\SDK\Exception\InvalidTokenException;
use Closure;

class CheckJWT
{
    protected $userRepository;

    /**
     * CheckJWT constructor.
     *
     * @param Auth0UserRepository $userRepository
     */
    public function __construct(Auth0UserRepository $userRepository)
    {
        $this->userRepository = $userRepository;
    }

    /**
     * Handle an incoming request.
     *
     * @param  \Illuminate\Http\Request  $request
     * @param  \Closure  $next
     * @return mixed
     */
    public function handle($request, Closure $next)
    {
        $auth0 = \App::make('auth0');

        $accessToken = $request->bearerToken() ?? "";
        try {
            $tokenInfo = $auth0->decodeJWT($accessToken);
            $user = $this->userRepository->getUserByDecodedJWT($tokenInfo);
            if (!$user) {
                return response()->json(["message" => "Unauthorized user"], 401);
            }

        } catch (InvalidTokenException $e) {
            return response()->json(["message" => $e->getMessage()], 401);
        } catch (CoreException $e) {
            return response()->json(["message" => $e->getMessage()], 401);
        }

        return $next($request);
    }
}

```

This middleware:

* Retrieves the Bearer token from the request, and calls the `decodeJWT` function with it to decode and verify the JWT.
* Uses the decoded and verified JWT to retrieve an `Auth0JWTUser`.
* Catches any exceptions thrown if the JWT is expired, malformed, or otherwise invalid.

Next, we register this middleware in the HTTP Kernel with the name `jwt`:

```php
// app/Http/Kernel.php
// ...
class Kernel extends HttpKernel {
	// ...
	protected $routeMiddleware = [
	    // ...
	    'jwt' => \App\Http\Middleware\CheckJWT::class,
	    // ...
	];
	// ...
}
```

We are now able to protect individual API endpoints by applying the `jwt` middleware:

```php
// routes/api.php

// This endpoint does not need authentication.
Route::get('/public', function (Request $request) {
    return response()->json(["message" => "Hello from a public endpoint! You don't need to be authenticated to see this."]);
});

// These endpoints require a valid access token.
Route::get('/private', function (Request $request) {
    return response()->json(["message" => "Hello from a private endpoint! You need to have a valid access token to see this."]);
})->middleware('jwt');
```

The `/api/private` route is now only accessible if a valid access token is included in the `Authorization` header of the incoming request. We can test this by manually generating an access token for the API and using a tool like Postman to test the routes.

In the Auth0 Dashboard, go to the **Test** tab for the API created above and click the **COPY TOKEN** link.

:::note
If you see a button to **Create & Authorize Test Application**, you'll need to click that before the **COPY TOKEN** button appears. After creating the test Application, click the **Machine to Machine Applications** tab, scroll down to the Application that was created, click the down icon, remove the `read:messages` permissions allowed for that Application, and click **Update**. Now click the **Test** tab, then **COPY TOKEN** to proceed.
:::

Now, let's turn on the Laravel test server:

```bash
php artisan serve --port=3010
```

Send a `GET` request to the public route  - `http://localhost:3010/api/public` - and you should receive back:

```json
{ "message": "Hello from a public endpoint!" }
```

Now send a `GET` request to the private route  - `http://localhost:3010/api/private` - and you should get a 401 status and the following message:

```json
{ "message": "Bearer token missing" }
```

Add an `Authorization` header set to `Bearer API_TOKEN_HERE` using the token generated above. Send the `GET` request to the private route again and you should see:

```json
{ "message": "Hello from a private endpoint!" }
```

### Configure the Scopes

The middleware we created above checks for the existence and validity of an access token but does not check the **scope** of the token. In this section, we will create another middleware to check for specific scopes.

Use the Artisan `make:middleware` again to create the middleware:

```bash
php artisan make:middleware CheckScope
```

Implement the `handle()` method that Laravel will call automatically for the route:

```php
// app/Http/Middleware/CheckScope.php

namespace App\Http\Middleware;

use Auth0\Login\Contract\Auth0UserRepository;
use Auth0\SDK\Exception\CoreException;
use Auth0\SDK\Exception\InvalidTokenException;
use Closure;

class CheckScope
{
    protected $userRepository;

    /**
     * CheckScope constructor.
     *
     * @param Auth0UserRepository $userRepository
     */
    public function __construct(Auth0UserRepository $userRepository)
    {
        $this->userRepository = $userRepository;
    }

    /**
     * Handle an incoming request.
     *
     * @param  \Illuminate\Http\Request  $request
     * @param  \Closure  $next
     * @param  \string  $scope
     * @return mixed
     */
    public function handle($request, Closure $next, $scope)
    {
        $auth0 = \App::make('auth0');

        $accessToken = $request->bearerToken() ?? "";
        try {
            $tokenInfo = $auth0->decodeJWT($accessToken);
            $user = $this->userRepository->getUserByDecodedJWT($tokenInfo);
            if (!$user) {
                return response()->json(["message" => "Unauthorized user"], 401);
            }

            if($scope) {
                $hasScope = false;
                if(isset($tokenInfo['scope'])) {
                    $scopes = explode(" ", $tokenInfo['scope']);
                    foreach ($scopes as $s) {
                        if ($s === $scope)
                            $hasScope = true;
                    }
                } 
                if(!$hasScope) {
                    return response()->json(["message" => "Insufficient scope"], 403);
                }
            }
        } catch (InvalidTokenException $e) {
            return response()->json(["message" => $e->getMessage()], 401);
        } catch (CoreException $e) {
            return response()->json(["message" => $e->getMessage()], 401);
        }

        return $next($request);
    }
}
```

This middleware:

* Similar to the `CheckJWT` middleware, it first uses the Bearer token from the request to decode and verify the JWT, handling any exceptions thrown as before.
* Verifies that the token contains the required `scope`.

Register this middleware in the HTTP Kernel with the name `check.scope`:

```php
// app/Http/Kernel.php
// ...
class Kernel extends HttpKernel {
	// ...
	protected $routeMiddleware = [
	    // ...
        'check.scope' => \App\Http\Middleware\CheckScope::class,
	    // ...
	];
	// ...
}
```

We are now able to protect individual API endpoints by applying the `check.scope` middleware:

```php
// routes/api.php
// ...

// These endpoints require a valid access token with a "read:messages" scope.
Route::get('/private-scoped', function (Request $request) {
    return response()->json([
        "message" => "Hello from a private endpoint! You need to have a valid access token and a scope of read:messages to see this."
    ]);
})->middleware('check.scope:read:messages');
```

This route is now only accessible if an access token used in the request has a scope of `read:messages`.

To test this route, first send a `GET` request with no token to the private, scoped route  - `http://localhost:3010/api/private-scoped` - and you should get a 401 status and the following message:

```json
{ "message": "Bearer token missing" }
```

Add an `Authorization` header set to `Bearer API_TOKEN_HERE` using the same token from the previous section. Send the `GET` request to the private, scoped route again and you should get a 403 status and the following message:

```json
{ "message": "Insufficient scope" }
```

Back in the Auth0 Dashboard:

1. Go to the **Machine to Machine Applications** tab for the API created above.
2. Scroll to the test Application for this API, make sure it's authorized, and click the down arrow icon.
3. Add the `read:messages` scope and click **Update** (then **Continue** if needed).

![](/media/articles/server-apis/authorize-m2m-for-api.png)

4. Click the **Test** tab, select the test Application, and click the **COPY TOKEN** link above the second code block.

Change the `Authorization` header to use the new token and send the `GET` request again. You should get a 200 status and the following message:

```json
{ "message": "Hello from a private, scoped endpoint!" }
```

## Obtaining Access Tokens

The example above uses manually-generated tokens which are not long-lived. Once your API is live on the web and ready to accept requests, the applications making the requests will need to get an access token for this API using one of a few ways:

- Web applications will use the [Authorization Code Flow](/flows/concepts/auth-code)
- Mobile, desktop, and other native applications will use a [Mobile/Native Login Flow](/flows/concepts/auth-code-pkce)
- CLIs, daemons, or services running on your back-end will use an [Machine-to-Machine Flow](/flows/concepts/client-credentials)

Regardless of the type, the application will need to request the audience of this API during the login flow to receive a correctly-formed access token.

## Configure CORS (optional)

To configure CORS, you should add the `laravel-cors` dependency. You can [check it out here](https://github.com/barryvdh/laravel-cors).

After installation, add `HandleCors` middleware in the application's global middleware stack:

```php
// app/Http/Kernel.php
protected $middleware = [
    // ...
    \Barryvdh\Cors\HandleCors::class,
];
```

Add the following to the configuration file for `CORS`:

```php
<?php
// config/cors.php
return [
    'supportsCredentials' => true,
    'allowedOrigins' => ['http://localhost:3000'],
    'allowedOriginsPatterns' => [],
    'allowedHeaders' => ['*'],
    'allowedMethods' => ['*'],
    'exposedHeaders' => [],
    'maxAge' => 0,
];
```
