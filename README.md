# Mail Login

Authenticate users through their email, 1 minute installation.

```html
<form method="post" action="/login/mail/send">
    @csrf
    <input type="email" name="email" placeholder="me@email.com">
    <button type="submit">Log in</button>
</form>
```

## [Download it](https://github.com/sponsors/DarkGhostHunter/sponsorships?sponsor=DarkGhostHunter&tier_id=195147&preview=false)

[![](sponsors.png)](https://github.com/sponsors/DarkGhostHunter/sponsorships?sponsor=DarkGhostHunter&tier_id=195147&preview=false)

[Become a Sponsor and get instant access to this package](https://github.com/sponsors/DarkGhostHunter/sponsorships?sponsor=DarkGhostHunter&tier_id=195147&preview=false).

## 1 minute quickstart

Login Mail is very simple to install: put the email of the user you want to authenticate in a form, and a mail will be sent to him with a link to authenticate.

First, publish the controllers to automatically log in the user. You will receive the `app\Http\Controllers\MailLogin\MailLoginController.php` file you can freely edit.

```shell
php artisan vendor:publish --provider="Laragear\MailLogin\MailLoginServiceProvider" --tag="controllers"
```

Then, register the routes in your application by just calling `MailLogin::routes()`, like inside your `web.php` routes file.

```php
use Illuminate\Routing\Route;
use Laragear\MailLogin\Facades\MailLogin;

Route::view('welcome');

// Register the default Mail Login routes
MailLogin::routes();
```

Finally, add a "login" box that receives the user email by making a `POST` to `login/mail/send`.

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
    $request->validate(['email' => 'required|email'])

    MailLogin::to($request->email);

    return back();
}
```

Since there may be times when the email of the user is not `email` but something else, like `mail` or `email_address`, you can change the email key using `emailIs()`.

```php
MailLogin::emailIs('email_address')->to(...);
```

### Specifying the guard

By default, is uses the application default guard, which in most _vanilla_ Laravel applications is `web`. You may want to change the default guard in the configuration, or change it at runtime using `guard()`:

```php
use Laragear\MailLogin\Facades\MailLogin;

MailLogin::guard('admins')->to(...);
```

### Idempotent email

To avoid sending multiple emails you may want to make the mailing _idempotent_, meaning, that subsequent requests to send the email will do nothing before a brief period of time.

Use the `idempotent()` method, optionally with the amount of seconds you want to keep the idempotency.

```php
use Laragear\MailLogin\Facades\MailLogin;

MailLogin::idempotent(60 * 3)->to(...);
```

> **Note**
> The idempotency key uses your application Rate Limiter. You can configure the rate limiting in the [config](#idempotency).

### URL

To change the login route the email should point to, use the `route()` method with the named route, and additional parameters if you want.

```php
use Laragear\MailLogin\Facades\MailLogin;

MailLogin::route('editor.login', ['theme' => 'blue'])->to(...);
```

> **Warning**
> The named route must exist. This route should show a form to login, **not** login the user immediately. See [Login in from a mail](#login-in-from-a-mail).

### Modifying the Mailable

The `to()` accepts a third argument as a Closure, which receives the `MailLogin` mailable. You're free to modify the mailable to your liking, like changing the view, or return your own mailable class.

```php
use Illuminate\Http\Request;
use Laragear\MailLogin\Facades\MailLogin;
use Laragear\MailLogin\Mails\LoginMail;

public function sendLoginMail(Request $request)
{
    $request->validate(['email' => 'required|email'])

    MailLogin::to($request->email, onMail: function (LoginMail $mailable) {
        $mailable->view('my-login-email', ['theme' => 'blue']);
        
        $mailable->subject('Login to this awesome app');
    });

    return back();
}
```

## Login in from a Mail

The Login is a two-part system: the user reaches the signed route with a form, and the form fires to authenticate the user. This is required because some **email clients will preload / cache / prefetch the login link**, usually to filter malicious links or cache assets. For us, this means to accidentally authenticate the user.

First, make a route that shows a form to login, and another to authenticate the user. The `mail-login::web-login` will take care to show the form, while the `LoginByMailRequest` will authenticate the user, so you will only need to return a redirection response. Both of these routes should be `signed` to avoid tampering with the query parameters, and share the same path.

```php
use Laragear\MailLogin\Http\Requests\LoginByMailRequest;
use Illuminate\Support\Facades\Route;

Route::get('login/mail', fn () => view('mail-login::web.login'))
    ->middleware(['guest', 'signed'])
    ->name('login.mail');

Route::post('login/mail', fn (LoginByMailRequest $request) => redirect('/dashboard'))
    ->middleware(['guest', 'signed']);
```

If you don't want to use the `LoginbyMailRequest` Request, you may log in the user manually:

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

This is the default Authentication Guard to use. When `null`, it fall backs to the application default, which is usually `web`. This is used to find user via the guard User Provider to login users. 

> **Info**
> When sending an email, the guard gets imprinted in the link to avoid changes on the configuration.

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

When mailing the link, a signed URL will be generated with an expiration time. You can control how many minutes to keep the link valid until it is detected as "expired" and no longer works.

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
        'prefix' => 'login-mail',
    ]
]
``` 

To avoid pushing multiple mails to your server, you may want to only send one mail inside a time window. For idempotency to work, it requires a prefix to avoid collisions with other rate limiters. The default string should be safe for most apps. 

## Laravel Octane Compatibility

* There are no singletons using a stale application instance.
* There are no singletons using a stale config instance.
* There are no singletons using a stale request instance.
* The only static property accessible to write is the `LoginByMailRequest::$destroyOnRegeneration`.

There should be no problems using this package with Laravel Octane.

## Security

If you discover any security related issues, please email darkghosthunter@gmail.com instead of using the issue tracker.

## License

The MIT License (MIT). Please see [License File](LICENSE.md) for more information.

Laravel is a Trademark of Taylor Otwell. Copyright ?? 2011-2022 Laravel LLC.
