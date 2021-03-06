# Roadmap - Socialite Login with Facebook

## Create project structure

```shell
laravel new Sociealite --jet
# Choose Inertia

cd Sociealite
npm install && npm run dev 
```mysql


## Change database host

- Open the `.env` file and set the host for mysql.

```dotenv
...
DB_HOST=mysql
...
```

## Run the migrations

```sh
sail artisan migrate
```

## Install Socialite Package in Laravel

```sh
sail composer require laravel/socialite
```

- Open `config/app.php`, register socialite plugin in providers, and aliases array.

```php

'providers' => [
    ....
    Laravel\Socialite\SocialiteServiceProvider::class,
],

'aliases' => [
    ....
    'Socialite' => Laravel\Socialite\Facades\Socialite::class,
],
```

## Add Facebook ID in Users Table

```sh
sail artisan make:migration add_fb_id_column_in_users_table --table=users
```

- Head over to newly created file `migrations/timestamp_add_fb_id_column_in_users_table.php` file, add the fb_id column value. 

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

class AddFbIdColumnInUsersTable extends Migration
{
    public function up(): void
    {
        Schema::table('users', function (Blueprint $table) {
            $table->string('fb_id')->nullable();
        });
    }

    public function down(): void
    {
        Schema::table('users', function (Blueprint $table) {
            $table->dropColumn('fb_id');
        });
    }
}
```

- Then migrate it

```sh
sail artisan migrate
```

- Also, add the Facebook ID table value in `app/Models/User.php` file.

```php
<?php

namespace App\Models;

use Illuminate\Contracts\Auth\MustVerifyEmail;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Laravel\Fortify\TwoFactorAuthenticatable;
use Laravel\Jetstream\HasProfilePhoto;
use Laravel\Sanctum\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens;
    use HasFactory;
    use HasProfilePhoto;
    use Notifiable;
    use TwoFactorAuthenticatable;

    /**
     * The attributes that are mass assignable.
     *
     * @var array
     */
    protected $fillable = [
        'name',
        'email',
        'password',
        'fb_id',
    ];

    /**
     * The attributes that should be hidden for arrays.
     *
     * @var array
     */
    protected $hidden = [
        'password',
        'remember_token',
        'two_factor_recovery_codes',
        'two_factor_secret',
    ];

    /**
     * The attributes that should be cast to native types.
     *
     * @var array
     */
    protected $casts = [
        'email_verified_at' => 'datetime',
    ];

    /**
     * The accessors to append to the model's array form.
     *
     * @var array
     */
    protected $appends = [
        'profile_photo_url',
    ];
}
```

## Add Facebook App ID and Secret

- In order to login with Facebook, we need to create a Facebook app that you can do by going to [Facebook Developer](https://developers.facebook.com/apps).

- After creating a Facebook app id and the secret app, register in the `config/services.php` file.

```php
return [
    ....
    'facebook' => [
        'client_id' => env('FACEBOOK_ID'),
        'client_secret' => env('FACEBOOK_SECRET'),
        'redirect' => env('APP_URL') . '/auth/facebook/callback',
    ],
]
```

## Set facebook environments

- Open the `.env` file and set the environments below:

```dotenv
...
FACEBOOK_ID=xxxxxx
FACEBOOK_SECRET=yyyyyy
...
```

## Generate and Configure Controller

```shell
sail artisan make:controller SocialController
```

- Open `app/Http/Controllers/SocialController.php` and place the following code.

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;

use App\Models\User;
use Validator;
use Socialite;
use Exception;
use Auth;

class SocialController extends Controller
{
    public function facebookRedirect()
    {
        return Socialite::driver('facebook')->redirect();
    }
    
    public function loginWithFacebook()
    {
        try {
    
            $user = Socialite::driver('facebook')->user();
            $isUser = User::where('fb_id', $user->id)->first();
     
            if($isUser){
                Auth::login($isUser);
                return redirect('/dashboard');
            }else{
                $createUser = User::create([
                    'name' => $user->name,
                    'email' => $user->email,
                    'fb_id' => $user->id,
                    'password' => encrypt('admin@123')
                ]);
    
                Auth::login($createUser);
                return redirect('/dashboard');
            }
    
        } catch (Exception $exception) {
            dd($exception->getMessage());
        }
    }
}
```

## Setting Up Route

- Open `routes/web.php` file and define the routes

```php
<?php

use Illuminate\Support\Facades\Route;
use App\Http\Controllers\SocialController;

/*
|--------------------------------------------------------------------------
| Web Routes
|--------------------------------------------------------------------------
|
| Here is where you can register web routes for your application. These
| routes are loaded by the RouteServiceProvider within a group which
| contains the "web" middleware group. Now create something great!
|
*/

Route::get('auth/facebook', [SocialController::class, 'facebookRedirect']);

Route::get('auth/facebook/callback', [SocialController::class, 'loginWithFacebook']);
```

## Add Login with Facebook Button in Login View

- Add Login with Facebook button in Login view template, so add the following code in `views/auth/login.blade.php` file.

```html
<x-guest-layout>
    <x-jet-authentication-card>
        <x-slot name="logo">
            <x-jet-authentication-card-logo />
        </x-slot>

        <x-jet-validation-errors class="mb-4" />

        @if (session('status'))
        <div class="mb-4 font-medium text-sm text-green-600">
            {{ session('status') }}
        </div>
        @endif

        <form method="POST" action="{{ route('login') }}">
            @csrf

            <div>
                <x-jet-label for="email" value="{{ __('Email') }}" />
                <x-jet-input id="email" class="block mt-1 w-full" type="email" name="email" :value="old('email')"
                    required autofocus />
            </div>

            <div class="mt-4">
                <x-jet-label for="password" value="{{ __('Password') }}" />
                <x-jet-input id="password" class="block mt-1 w-full" type="password" name="password" required
                    autocomplete="current-password" />
            </div>

            <div class="block mt-4">
                <label for="remember_me" class="flex items-center">
                    <input id="remember_me" type="checkbox" class="form-checkbox" name="remember">
                    <span class="ml-2 text-sm text-gray-600">{{ __('Remember me') }}</span>
                </label>
            </div>

            <div class="flex items-center justify-end mt-4">
                @if (Route::has('password.request'))
                <a class="underline text-sm text-gray-600 hover:text-gray-900" href="{{ route('password.request') }}">
                    {{ __('Forgot your password?') }}
                </a>
                @endif

                <x-jet-button class="ml-4">
                    {{ __('Login') }}
                </x-jet-button>
            </div>

            {{-- Login with Facebook --}}
            <div class="flex items-center justify-end mt-4">
                <a class="btn" href="{{ url('auth/facebook') }}"
                    style="background: #3B5499; color: #ffffff; padding: 10px; width: 100%; text-align: center; display: block; border-radius:3px;">
                    Login with Facebook
                </a>
            </div>
        </form>
    </x-jet-authentication-card>
</x-guest-layout>
```

## Start the application.

```shell
sail artisan serve

# http://localhost/login
```


## References

- https://www.positronx.io/laravel-socialite-login-with-facebook-tutorial-with-example/
