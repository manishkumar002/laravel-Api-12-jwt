# What is an API?
An API (Application Programming Interface) is a set of rules that allows different software applications to communicate with each other. APIs enable the backend server to provide data and services to frontend clients, such as websites or mobile apps.

**What is JWT?**

JSON Web Token (JWT) is a standard for securely transmitting information as a JSON object. It is self-contained, compact, and digitally signed. JWTs are commonly used to authenticate users in web and mobile applications, allowing the server to verify a user's identity without storing session information.

**Why Use JWT in Laravel?**

Stateless authentication — no need to store session data on the server.

# Scalable and suitable for APIs.

Can be used across different domains and platforms.

### Secure transmission of user info.

**Tutorial Overview**

We will build a RESTful API in Laravel 12 with the following endpoints:

**Endpoint	Method	Description**

/api/auth/register	POST	Register a new user
/api/auth/login	POST	User login and get JWT
/api/auth/profile	POST	Get authenticated user info
/api/auth/refresh	POST	Refresh the JWT token
/api/auth/logout	POST	Logout user and invalidate token

**Step 1: Install Laravel 12**

If you haven't already, install Laravel 12 via Composer:

composer create-project laravel/laravel example-app
cd example-app

**Step 2: Enable API and Customize Authentication Exception**

Laravel 12’s API routes need to be enabled. You can do this by running:

**php artisan install:api**

Next, update the bootstrap/app.php file to customize the authentication exception response for API routes so unauthorized access returns a JSON response:

```base
<?php

use Illuminate\Foundation\Application;
use Illuminate\Foundation\Configuration\Exceptions;
use Illuminate\Foundation\Configuration\Middleware;
use Illuminate\Auth\AuthenticationException;
use Illuminate\Http\Request;

return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        api: __DIR__.'/../routes/api.php',
        commands: __DIR__.'/../routes/console.php',
        health: '/up',
    )
    ->withMiddleware(function (Middleware $middleware) {
        //
    })
    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->render(function (AuthenticationException $e, Request $request) {
            if ($request->is('api/*')) {
                return response()->json([
                    'message' => $e->getMessage(),
                ], 401);
            }
        });
    })->create();
```

**Step 3: Install JWT Package**
Install the php-open-source-saver/jwt-auth package:

composer require php-open-source-saver/jwt-auth
Publish the package configuration:

```base
php artisan vendor:publish --provider="PHPOpenSourceSaver\JWTAuth\Providers\LaravelServiceProvider"
```

Generate the JWT secret key (add to .env):

```base
php artisan jwt:secret
```

Open .env and set:

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=laravel_jwt
DB_USERNAME=root
DB_PASSWORD=
(Change DB settings if needed)

Run the migration command
```base
php artisan migrate
```
**Step 4: Configure Authentication Guard**
Open config/auth.php and update the api guard to use jwt driver:

```base
'guards' => [
    // other guards...

    'api' => [
        'driver' => 'jwt',
        'provider' => 'users',
        'hash' => false,
    ],
],
```
**Step 5: Update User Model**

Edit app/Models/User.php to implement JWTSubject:

```base
<?php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use PHPOpenSourceSaver\JWTAuth\Contracts\JWTSubject;

class User extends Authenticatable implements JWTSubject
{
    protected $fillable = ['name', 'email', 'password'];
    protected $hidden = ['password', 'remember_token'];

    public function getJWTIdentifier()
    {
        return $this->getKey();
    }

    public function getJWTCustomClaims()
    {
        return [];
    }
}
```
**Step 6: Create API Routes**

Define API routes in routes/api.php:

```base
use Illuminate\Support\Facades\Route;
use App\Http\Controllers\API\AuthController;

Route::prefix('auth')->group(function () {
    Route::post('register', [AuthController::class, 'register']);
    Route::post('login', [AuthController::class, 'login']);

    Route::middleware('auth:api')->group(function () {
        Route::post('profile', [AuthController::class, 'profile']);
        Route::post('logout', [AuthController::class, 'logout']);
        Route::post('refresh', [AuthController::class, 'refresh']);
    });
});
```
**Step 7: Create AuthController**
Create an API controller to handle user registration, login, logout, and token refresh.

Run:

```base
php artisan make:controller API/AuthController

Then, add the following code inside app/Http/Controllers/API/AuthController.php:

<?php

namespace App\Http\Controllers\API;

use App\Http\Controllers\Controller;
use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Validator;

class AuthController extends Controller
{
    // Register new user
    public function register(Request $request)
    {
        $validator = Validator::make($request->all(), [
            'name'       => 'required|string|max:255',
            'email'      => 'required|string|email|max:255|unique:users',
            'password'   => 'required|string|min:6|confirmed',
        ]);

        if ($validator->fails()) {
            return response()->json(['errors' => $validator->errors()], 422);
        }

        $user = User::create([
            'name'     => $request->name,
            'email'    => $request->email,
            'password' => bcrypt($request->password),
        ]);

        return response()->json(['message' => 'User registered successfully', 'user' => $user], 201);
    }

    // Login user and return JWT token
    public function login(Request $request)
    {
        $credentials = $request->only('email', 'password');

        if (!$token = auth('api')->attempt($credentials)) {
            return response()->json(['error' => 'Unauthorized'], 401);
        }

        return $this->respondWithToken($token);
    }

    // Get user profile
    public function profile()
    {
        return response()->json(auth('api')->user());
    }

    // Logout user (invalidate token)
    public function logout()
    {
        auth('api')->logout();

        return response()->json(['message' => 'Successfully logged out']);
    }

    // Refresh JWT token
    public function refresh()
    {
        return $this->respondWithToken(auth('api')->refresh());
    }

    // Return token response structure
    protected function respondWithToken($token)
    {
        return response()->json([
            'access_token' => $token,
            'token_type'   => 'bearer',
            'expires_in'   => auth('api')->factory()->getTTL() * 60,
        ]);
    }
}
```
**Step 8: Run Laravel Server**

```base
php artisan serve
```
Your API will run at http://localhost:8000.

## How to Use the API

**1. Register User**
URL: POST /api/auth/register

```base
Body:

{
  "name": "StarCode Kh",
  "email": "starcodekh@example.com",
  "password": "12345678",
  "password_confirmation": "12345678"
}
```
**2. Login User**

URL: POST /api/auth/login

```base
Body:

{
  "email": "starcodekh@example.com",
  "password": "12345678"
}
Response:

{
  "access_token": "token_here",
  "token_type": "bearer",
  "expires_in": 3600
}
```
**3. Get Profile**

URL: POST /api/auth/profile

Headers: Authorization: Bearer {access_token}

**4. Refresh Token**

URL: POST /api/auth/refresh

Headers: Authorization: Bearer {access_token}

**5. Logout**

URL: POST /api/auth/logout

Headers: Authorization: Bearer {access_token}

## Conclusion
You have now built a secure JWT authentication API using Laravel 12! This setup is ideal for web or mobile apps that require stateless authentication.

You can extend this tutorial by adding roles, permissions, or social logins.

Want the full source code?

Download the complete Laravel 12 JWT API Authentication example on my GitHub repo here.
