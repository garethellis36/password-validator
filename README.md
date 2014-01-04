# Password Validator [![Build Status](https://travis-ci.org/jeremykendall/password-validator.png?branch=master)](https://travis-ci.org/jeremykendall/password-validator)

**Password Validator** *validates* `password_hash` generated passwords, *rehashes*
passwords as necessary, and will *upgrade* legacy passwords.

## Motivation

Why? Because one must always[ encrypt passwords for highest level of
security][7], and the new [PHP password hashing][1] functions provide that level of
security.

The **Password Validator** library makes it (more) trivial to use the new
password hash functions in your application.  Just add the validator to your
authentication script and you're up and running.

The really big deal here is the **ease of upgrading** from your current legacy
hashes to the new, more secure PHP password hash hashes.  Simply wrap the
`PasswordValidator` in the `UpgradeDecorator`, provide a callback to validate
your existing password hashing scheme, and BOOM, you're using new password
hashes in a manner *completely transparent* to your application's users. Nifty,
huh?

## Usage

### Password Validation

If you're already using [`password_hash`][2] generated passwords in your
application, you need do nothing more than add the validator in your
authentication script. The validator uses [`password_verify`][3] to test 
the validity of the provided password hash.

``` php
use JeremyKendall\Password\PasswordValidator;

$validator = new PasswordValidator();
$result = $validator->isValid($_POST['password'], $hashedPassword);

if ($result->isValid()) {
    // password is valid
}
```

If your application requires options other than the `password_hash` defaults,
you can set both the `salt` and `cost` options with `PasswordValidator::setOptions()`.

``` php
$options = array(
    'salt' => 'SettingYourOwnSaltIsNotTheBestIdea',
    'cost' => 11,
);
$validator->setOptions($options);
```

**IMPORTANT**: If you're using a `cost` other than the default cost of `10`,
your passwords will be rehashed with a cost of `10` *unless* you set the cost
using `PasswordValidator::setOptions()`.

#### Validation Result

Each validation attempt returns a `Result` object. The object provides some
introspection into the status of the validation process.

* `Result::isValid()` will return `true` if the attempt was successful
* `Result::getCode()` will return one of three possible `int` codes:
    * `Result::SUCCESS` if the validation attempt was successful
    * `Result::SUCCESS_PASSWORD_REHASHED` if the attempt was successful and the password was rehashed
    * `Result::FAILURE_PASSWORD_INVALID` if the attempt was unsuccessful
* `Result::getPassword()` will return the rehashed password, but only if the password was rehashed

### Rehashing

Each valid password is tested using [`password_needs_rehash`][4]. If a rehash
is necessary, the valid password is hashed using `password_hash` with the
provided options. The result code `Result::SUCCESS_PASSWORD_REHASHED` will be
returned from `Result::getCode()` and the new password hash is available via
`Result::getPassword()`.

``` php
if ($result->isValid() && $result->getCode() == Result::SUCCESS_PASSWORD_REHASHED) {
    $rehashedPassword = $result->getPassword();
    // Persist rehashed password
}
```

**IMPORTANT**: If the password has been rehashed, it's critical that you
persist the updated password hash. Otherwise, what's the point, right?

### Upgrading Legacy Passwords

You can use the `PasswordValidator` whether or not you're currently using
`password_hash` generated passwords. The validator will upgrade your current
legacy hashes to the new `password_hash` generated hashes.  All you need to
do is provide a validator callback for your password hash and then
[decorate][6] the validator with the `UpgradeDecorator`.

``` php
use JeremyKendall\Password\Decorator\UpgradeDecorator;

// Example callback to validate a sha512 hashed password
$callback = function ($password, $passwordHash) {
    if (hash('sha512', $password) === $passwordHash) {
        return true;
    }

    return false;
};

$validator = new UpgradeDecorator(new PasswordValidator(), $callback);
```

The `UpgradeDecorator` will validate a user's current password using the
callback validator.  If the user's password is valid, it will be hashed with
`password_hash` and returned in the `Result` object, as above.

If the callback determines the password is invalid, the password will be passed
along to the `PasswordValidator` in case it's already been upgraded.

### Database Schema Changes

**IMPORTANT**: Because this library uses the `PASSWORD_DEFAULT` algorithm, it's
important your password field be `VARCHAR(255)` to account for future updates
to the default password hashing algorithm.

## Helper Scripts

After running `composer install`, there are two helper scripts available, both 
related to the password hash functions.

### version-check

If you're not already running PHP 5.5+, you should run `version-check` to
ensure your version of PHP is capable of using password-compat, the userland
implementation of the PHP password hash functions.  Run `./bin/version-check`
from the root of your project. The result of the script is pass/fail.

### cost-check

The default `cost` used by `password_hash` is 10.  This may or may not be
appropriate for your production hardware, and it's entirely likely you can use
a higher cost than the default. `cost-check` is based on the [finding a good
cost][8] example in the PHP documentation. Simply run `./bin/cost-check` from the command line and an appropriate cost will be returned.

**NOTE**: The default time target is 0.2 seconds.  You may choose a higher or lower 
target by passing a float argument to `cost-check`, like so:

``` bash
$ ./bin/cost-check 0.4
Appropriate 'PASSWORD_DEFAULT' Cost Found:  13
```

## Installation

The only officially supported method of installation is via
[Composer](http://getcomposer.org).

Create a `composer.json` file in the root of your project (always check
Packagist for the most recent version):

``` json
{
    "require": {
        "jeremykendall/password-validator": "*"
    }
}
```

And then run: `composer install`

Add the autoloader to your project:

``` php
<?php

require_once '../vendor/autoload.php'
```

You're now ready to begin using the Password Validator.

## Contributing

Pull requests are *always* welcome. Please review the CONTRIBUTING.md document before
submitting pull requests.

[1]: http://www.php.net/manual/en/ref.password.php
[2]: http://www.php.net/manual/en/function.password-hash.php
[3]: http://www.php.net/manual/en/function.password-verify.php
[4]: http://www.php.net/manual/en/function.password-needs-rehash.php
[5]: https://github.com/ircmaxell/password_compat
[6]: http://en.wikipedia.org/wiki/Decorator_pattern
[7]: http://csiphp.com/blog/2012/02/16/encrypt-passwords-for-highest-level-of-security/
[8]: http://php.net/password_hash#example-875
