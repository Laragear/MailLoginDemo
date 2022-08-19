# Login Mail

Authenticate users through their email, no password needed.

```php
use Laragear\MailLogin\Facades\MailLogin;
use Laragear\MailLogin\Http\Requests\LoginByMailRequest;
use Illuminate\Http\Request;

public function sendLoginMail(Request $request)
{
    MailLogin::to($request->email);
	
    return 'Check your email with a link to login!';
}

public function login(LoginByMailRequest $request)
{
    return "Welcome back, {$request->user()->name}";
}
```

## [This private package is only available for Sponsors](https://github.com/sponsors/DarkGhostHunter/sponsorships?sponsor=DarkGhostHunter&tier_id=195147&preview=false)

Become one and get instant access [by signing up here](https://github.com/sponsors/DarkGhostHunter/sponsorships?sponsor=DarkGhostHunter&tier_id=195147&preview=false)

## 5 minutes quickstart

Login Mail is very simple to install: put the email of the user you want to authenticate in a form, and a mail will be sent to him with a link to authenticate.

First, publish the controllers to automatically log in the user. You will receive the `app\Http\Controllers\MailLogin\MailLoginController.php` file you can freely edit.

```shell
php artisan vendor:publish --provider="Laragear\LoginMail\LoginMailServiceProvider" --tag="controllers"
```

Then, register the routes in your application by just calling `MailLogin::routes()`, like inside your `web.php` routes file.

```php
use Illuminate\Routing\Route;
use Laragear\MailLogin\Facades\MailLogin;

Route::view('welcome');

// Register the default Passmail routes
MailLogin::routes();
```

Finally, add a "login" box that asks the user its email and sends an email by posting it to `login/mail/send`.

```html
<form method="post" action="/login/mail/send">
    @csrf
    <input type="email" name="email" placeholder="me@email.com">
    <button type="submit">Log in</button>
</form>
```

This package will handle sending the mail and the authentication view for you.

## Sending the login email

To send an email manually, use the `MailLogin` facade with the user credentials. You can also use an email string.

```php
use Illuminate\Http\Request;
use Laragear\MailLogin\Facades\MailLogin;

public function sendLoginMail(Request $request)
{
    $request->validate(['email' => 'required|email|exists:users']);

    return back();
}
```

Since there may be times when the email of the user is not `email` but something else, like `mail` or `email_address`, you can change the email key using `emailIs()`.

```php
MailLogin::emailIs('email_address')->to(...);
```

### Specifying the guard

By default, is uses the application default guard. You may want to change the default guard in the configuration, or change it at runtime using `guard()`:

```php
use Laragear\MailLogin\Facades\MailLogin;

MailLogin::guard('admins')->to(...);
```

### Idempotent email

To avoid using a Rate Limiter, or sending multiple emails, you may want to make the mailing idempotent, meaning, that subsequent requests to send the email will do nothing before a brief period of time.

Use the `idempotent()` method, optionally with the amount of seconds you want to keep the idempotency.

```php
use Laragear\MailLogin\Facades\MailLogin;

MailLogin::idempotent(60 * 3)->to(...);
```

> The idempotency key uses your default cache. You can change the cache in the [config](#idempotency).

### URL

To change the login route the email should point to, use the `route()` method with the named route, and additional parameters if you want.

```php
use Laragear\MailLogin\Facades\MailLogin;

MailLogin::route('editor.login', ['theme' => 'blue'])->to(...);
```

> The named route must exist. This route should show a form to login, **not** login the user immediately. See [Login in from a mail](#login-in-from-a-mail).

## Login in from a Mail

The Login is a two-part system: the user reaches the signed route to press a button, and a secondary controller action authenticates the user. Additional user interactivity is required because some **email clients will preload / cache / prefetch the login link**. This happens to filter malicious links to other sites, but for us, it would authenticate the user by accident.

First, make a route that shows a form to login, and another to authenticate the user. The `mail-login::web-login` will take care to show the form, while the `LoginByMailRequest` will properly authenticate the user automatically so you will only need to return a redirection response.

```php
use Laragear\MailLogin\Http\Requests\LoginByMailRequest;
use Illuminate\Support\Facades\Route;

Route::get('login/mail', fn () => view('mail-login::web.login'))
    ->middleware(['guest', 'signed'])
    ->name('login.mail');

Route::post('login/mail', fn (LoginByMailRequest $request) => redirect('/dashboard'))
    ->middleware(['guest', 'signed']);
```

Otherwise, you may log in the user manually:

```php
use Illuminate\Support\Facades\Route;
use Illuminate\Support\Facades\Auth;
use Illuminate\Http\Request;

Route::post('login/mail', function (Request $request) {
    $request->validate(['id' => 'required|numeric']);

    Auth::loginUsingId($request->input('id'));

    $request->session()->regenerate();

    return redirect('/dashboard');
})->middleware(['guest', 'signed']);
```

## Advanced Configuration

Mail Login was made to work out-of-the-box, but you can override the configuration by simply publishing the config file.

```shell
php artisan vendor:publish --provider="Laragear\MailLogin\MailLoginServiceProvider" --tag="config"
```

After that, you will receive the `config/mail-login.php` config file with an array like this:

```php
return [
    'guard' => null,
    'route' => [
        'name' => 'login.mail',
        'view' => 'mail-login::web.login',
    ],
    'minutes' => 5,
    'mail' => [
        'mailer' => null,
        'connection' => null,
        'queue' => null,
        'view' => 'mail-login::mail.login',
    ],
    'idempotency' => [
        'store' => null,
        'prefix' => 'mail-login',
    ]
];
```

### Guard

```php
return [
    'guard' => null,
];
```

This is the default Authentication Guard to use. When `null`, it fall backs to the application default, which is usually `web`. This used to find user via the guard User Provider, and login users automatically.

### Route name & View

```php
return [
    'route' => [
        'name' => 'login.mail',
        'view' => 'mail-login::web.login',
    ],
];
```

This named route is linked in the email, which contains the view form to log in the user. We won't log him in directly because some mail clients will prefetch / preload the login link and may log him in by accident.

### Minutes to expire

```php
return [
    'minutes' => 5,
];
```

When mailing the link, a signed URL will be generated with an expiration time. You can control how many minutes to keep the link valid until it is detected in the included view form as "expired" and no longer working.

### Mail driver

```php
return [
    'mail' => [
        'mailer' => null,
        'connection' => null,
        'queue' => null,
        'view' => 'mail-login::mail.login',
    ],
];
```

This specifies which mail driver to use to send the login email, and the queue connection and name that will receive it. When `null`, it will fall back to the application default, which is usually `smtp`.

### Idempotency

```php
return [
    'idempotency' => [
        'store' => null,
        'prefix' => 'login-mail',
    ]
]
``` 

To avoid pushing multiple mails to your server, you may want to only send one mail inside a time window. For idempotency to work, it requires the cache. Here are the cache to use (if not default) and the key prefix.

## Laravel Octane Compatibility

* There are no singletons using a stale application instance.
* There are no singletons using a stale config instance.
* There are no singletons using a stale request instance.
* There are no static properties written during a request.

There should be no problems using this package with Laravel Octane.

## Security

If you discover any security related issues, please email darkghosthunter@gmail.com instead of using the issue tracker.

## License

The MIT License (MIT). Please see [License File](LICENSE.md) for more information.

Laravel is a Trademark of Taylor Otwell. Copyright © 2011-2022 Laravel LLC.
