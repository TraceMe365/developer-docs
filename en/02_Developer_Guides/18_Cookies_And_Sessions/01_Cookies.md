---
title: Cookies
summary: A set of static methods for manipulating PHP cookies.
icon: cookie-bite
---

# Cookies

Note that cookies can have security implications - before setting your own cookies, make sure to read through the
[secure coding](/developer_guides/security/secure_coding#secure-sessions-cookies-and-tls-https) documentation.

## Accessing and manipulating cookies

Cookies are a mechanism for storing data in the remote browser and thus tracking or identifying return users.

Silverstripe CMS uses cookies for remembering users preferences. Application code can modify a users cookies through
the [Cookie](api:SilverStripe\Control\Cookie) class. This class mostly follows the PHP API.

### Set

Sets the value of cookie with configuration.

```php
use SilverStripe\Control\Cookie;

Cookie::set($name, $value, $expiry = 90, $path = null, $domain = null, $secure = false, $httpOnly = false);

// Cookie::set('MyApplicationPreference', 'Yes');
```

[info]
To set a cookie for less than 1 day, you can assign an `$expiry` value that is lower than 1. e.g. `Cookie::set('name', 'value', $expiry = 0.5);` will set a cookie for 12 hours.
[/info]

### Get

Returns the value of cookie.

```php
Cookie::get($name);

// Cookie::get('MyApplicationPreference');
// returns 'Yes'
```

### Force_expiry

Clears a given cookie.

```php
Cookie::force_expiry($name, $path = null, $domain = null);

// Cookie::force_expiry('MyApplicationPreference')
```

### Samesite attribute

The `samesite` attribute is set on all cookies with a default value of `Lax`. You can change the default value by setting the `default_samesite` value on the
[Cookie](api:SilverStripe\Control\Cookie) class:

```yml
SilverStripe\Control\Cookie:
  default_samesite: 'Strict'
```

[info]
Note that this *doesn't* apply for the session cookie, which is handled separately. See [Sessions](/developer_guides/cookies_and_sessions/sessions#samesite-attribute).
[/info]

## Cookie_Backend

The [Cookie](api:SilverStripe\Control\Cookie) class manipulates and sets cookies using a [Cookie_Backend](api:SilverStripe\Control\Cookie_Backend). The backend is in charge of the logic
that fetches, sets and expires cookies. By default we use a [CookieJar](api:SilverStripe\Control\CookieJar) backend which uses PHP's
[setcookie](https://www.php.net/manual/en/function.setcookie.php) function.

The [CookieJar](api:SilverStripe\Control\CookieJar) keeps track of cookies that have been set by the current process as well as those that were received
from the browser.

```php
use SilverStripe\Control\Cookie;
use SilverStripe\Control\CookieJar;
use SilverStripe\Core\Injector\Injector;

$myCookies = [
    'cookie1' => 'value1',
];

$newBackend = new CookieJar($myCookies);

Injector::inst()->registerService($newBackend, 'Cookie_Backend');

Cookie::get('cookie1');
```

### Resetting the cookie_backend state

Assuming that your application hasn't messed around with the `$_COOKIE` superglobal, you can reset the state of your
`Cookie_Backend` by simply unregistering the `CookieJar` service with `Injector`. Next time you access `Cookie` it'll
create a new service for you using the `$_COOKIE` superglobal.

```php
Injector::inst()->unregisterNamedObject('Cookie_Backend');

// will return $_COOKIE['cookiename'] if set
Cookie::get('cookiename');
```

Alternatively, if you know that the superglobal has been changed (or you aren't sure it hasn't) you can attempt to use
the current `CookieJar` service to tell you what it was like when it was registered.

```php
//store the cookies that were loaded into the `CookieJar`
$recievedCookie = Cookie::get_inst()->getAll(false);

//set a new `CookieJar`
Injector::inst()->registerService(new CookieJar($recievedCookie), 'CookieJar');
```

### Using your own cookie_backend

If you need to implement your own Cookie_Backend you can use the injector system to force a different class to be used.

```yml
---
Name: mycookie
After: '#cookie'
---
SilverStripe\Core\Injector\Injector:
  Cookie_Backend:
    class: MyCookieJar
```

To be a valid backend your class must implement the [Cookie_Backend](api:SilverStripe\Control\Cookie_Backend) interface.

## Advanced usage

### Sent vs received cookies

Sometimes it's useful to be able to tell if a cookie was set by the process (thus will be sent to the browser) or if it
came from the browser as part of the request.

Using the `Cookie_Backend` we can do this like such:

```php
Cookie::set('CookieName', 'CookieVal');

//gets the cookie as we set it
Cookie::get('CookieName');

//will return the cookie as it was when it was sent in the request
Cookie::get('CookieName', false);
```

### Accessing all the cookies at once

One can also access all of the cookies in one go using the `Cookie_Backend`

```php
//returns all the cookies including ones set during the current process
Cookie::get_inst()->getAll();

//returns all the cookies in the request
Cookie::get_inst()->getAll(false);
```

## API documentation

- [Cookie](api:SilverStripe\Control\Cookie)
- [CookieJar](api:SilverStripe\Control\CookieJar)
- [Cookie_Backend](api:SilverStripe\Control\Cookie_Backend)
