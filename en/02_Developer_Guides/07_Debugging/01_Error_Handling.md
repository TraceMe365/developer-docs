---
title: Logging and Error Handling
summary: Trap, fire and report diagnostic logs, user exceptions, warnings and errors.
icon: exclamation-circle
---

# Logging and error handling

Silverstripe CMS uses Monolog for both error handling and logging. It comes with two default configurations: one for
logging, and another for core error handling. The core error handling implementation also comes with two default
configurations: one for development environments, and another for test or live environments. On development
environments, Silverstripe CMS will deal harshly with any warnings or errors: a full call-stack is shown and execution
stops for anything, giving you early warning of a potential issue to handle.

[info]
There are a range of monolog handlers available, both in the core package and in add-ons. See the
[Monolog documentation](https://github.com/Seldaek/monolog/blob/main/doc/01-usage.md) for more information.
[/info]

## Raising errors and logging diagnostic information

For general purpose logging, you can use the Logger directly. The Logger is a PSR-3 compatible LoggerInterface and
can be accessed via the `Injector`:

```php
use Psr\Log\LoggerInterface;
use SilverStripe\Core\Injector\Injector;
use SilverStripe\Security\Security;

Injector::inst()->get(LoggerInterface::class)->info('User has logged in: ID #' . Security::getCurrentUser()->ID);
Injector::inst()->get(LoggerInterface::class)->debug('Query executed: ' . $sql);
Injector::inst()->get(LoggerInterface::class)->error('Something went wrong, but let\'s continue on...');
```

Although you can raise more important levels of alerts in this way, we recommend using PHP's native error systems for
these instead.

For notice-level and warning-level issues, you can also use [user_error](https://www.php.net/user_error) to throw errors
where appropriate. As with the default Logger implementation these will not halt execution, but will send a message
to the PHP error log.

```php
namespace App\Model;

use SilverStripe\ORM\DataObject;

class MyObject extends DataObject
{
    // ...

    public function delete()
    {
        if ($this->alreadyDelete) {
            user_error('Delete called on already deleted object', E_USER_NOTICE);
            return;
        }
        // ...
    }

    public function getRelatedObject()
    {
        if (!$this->RelatedObjectID) {
            user_error("Can't find a related object", E_USER_WARNING);
            return;
        }
        // ...
    }
}
```

For errors that should halt execution, you should use Exceptions. Normally, Exceptions will halt the flow of execution,
but they can be caught with a try/catch clause.

```php
throw new LogicException('Query failed: ' . $sql);
```

### Accessing the logger via dependency injection

It can be quite verbose to call `Injector::inst()->get(LoggerInterface::class)` all the time. More importantly,
it also means that you're coupling your code to global state, which is a bad design practise. A better
approach is to use dependency injection to pass the logger in for you. The [Injector](../extending/Injector)
can help with this. The most straightforward is to specify a `dependencies` config setting, like this:

```php
namespace App\Control;

use Psr\Log\LoggerInterface;
use SilverStripe\Control\Controller;

class MyController extends Controller
{
    private static $dependencies = [
        'Logger' => '%$' . LoggerInterface::class,
    ];

    /**
     * This will be set automatically, as long as MyController is instantiated via Injector
     *
     * @var LoggerInterface
     */
    protected $logger;

    protected function init()
    {
        $this->logger->debug('MyController::init() called');
        parent::init();
    }

    /**
     * @param LoggerInterface $logger
     * @return $this
     */
    public function setLogger(LoggerInterface $logger)
    {
        $this->logger = $logger;
        return $this;
    }
}
```

In other contexts, such as testing or batch processing, logger can be set to a different value by the code calling
MyController.

### Error levels

- **E_USER_WARNING:** Err on the side of over-reporting warnings. Throwing warnings provides a means of ensuring that
developers know:
  - Deprecated functions / usage patterns
  - Strange data formats
  - Things that will prevent an internal function from continuing.  Throw a warning and return null.

- **E_USER_ERROR:** Throwing one of these errors is going to take down the production site.  So you should only throw
E_USER_ERROR if it's going to be **dangerous** or **impossible** to continue with the request. Note that it is
preferable to now throw exceptions instead of `E_USER_ERROR`.

## Configuring error logging

You can configure your logging using Monolog handlers. The handlers should be provided in the `Logger.handlers`
configuration setting. Below we have a couple of common examples, but Monolog comes with [many different handlers](https://github.com/Seldaek/monolog/blob/main/doc/02-handlers-formatters-processors.md#handlers)
for you to try.

### Sending emails

To send emails, you can use Monolog's `NativeMailerHandler`, like this:

```yml
SilverStripe\Core\Injector\Injector:
  Psr\Log\LoggerInterface:
    calls:
      MailHandler: [ pushHandler, [ '%$MailHandler' ] ]
  MailHandler:
      class: Monolog\Handler\NativeMailerHandler
      constructor:
        - me@example.com
        - There was an error on your test site
        - me@example.com
        - error
      properties:
        ContentType: text/html
        Formatter: '%$SilverStripe\Logging\DetailedErrorFormatter'
```

The first section 4 lines passes a new handler to `Logger::pushHandler()` from the named service `MailHandler`. The
next 10 lines define what the service is.

The calls key, `MailHandler`, can be anything you like: its main purpose is to let other configuration disable it
(see below).

### Logging to a file

To log to a file, you can use Monolog's `StreamHandler`, like this:

```yml
SilverStripe\Core\Injector\Injector:
  Psr\Log\LoggerInterface:
    calls:
      LogFileHandler: [ pushHandler, [ '%$LogFileHandler' ] ]
  LogFileHandler:
    class: Monolog\Handler\StreamHandler
    constructor:
      - "/var/www/silverstripe.log"
      - "info"
```

[warning]
The log file path must be an absolute file path, as relative paths may behave differently between CLI and HTTP requests. If you want to use a *relative* path, you can use the `SS_ERROR_LOG` environment variable to declare a file path that is relative to your project root:

```bash
SS_ERROR_LOG="./silverstripe.log"
```

You don't need any of the YAML configuration above if you are using the `SS_ERROR_LOG` environment variable - but you can use a combination of the environment variable and YAML configuration if you want to configure multiple error log files.
[/warning]

[notice]
You will need to make sure the user running the PHP process has write access to the log file, wherever you choose to put it.
[/notice]

The `info` argument provides the minimum level to start logging at.

### Disabling the default handler

You can disable a handler by removing its pushHandlers call from the calls option of the Logger service definition.
The handler key of the default handler is `pushDisplayErrorHandler`, so you can disable it like this:

```yml
SilverStripe\Core\Injector\Injector:
  Psr\Log\LoggerInterface.errorhandler:
    calls:
      pushDisplayErrorHandler: '%%remove%%'
```

### Setting a different configuration for dev

In order to set different logging configuration on different environment types, we rely on the environment-specific
configuration features that the config system providers. For example, here we have different configuration for dev and
non-dev.

```yml
---
Name: dev-errors
Only:
  environment: dev
---
SilverStripe\Core\Injector\Injector:
  Psr\Log\LoggerInterface.errorhandler:
    calls:
      pushMyDisplayErrorHandler: [ pushHandler, [ '%$DisplayErrorHandler' ]]
  DisplayErrorHandler:
    class: SilverStripe\Logging\HTTPOutputHandler
    constructor:
      - "notice"
    properties:
      Formatter: '%$SilverStripe\Logging\DetailedErrorFormatter'
      CLIFormatter: '%$SilverStripe\Logging\DetailedErrorFormatter'
---
Name: live-errors
Except:
  environment: dev
---
SilverStripe\Core\Injector\Injector:
  # Default logger implementation for general purpose use
  Psr\Log\LoggerInterface:
    calls:
      # Save system logs to file
      pushFileLogHandler: [ pushHandler, [ '%$LogFileHandler' ]]

  # Core error handler for system use
  Psr\Log\LoggerInterface.errorhandler:
    calls:
      # Save errors to file
      pushFileLogHandler: [ pushHandler, [ '%$LogFileHandler' ]]
      # Format and display errors in the browser/CLI
      pushMyDisplayErrorHandler: [ pushHandler, [ '%$DisplayErrorHandler' ]]

  # Custom handler to log to a file
  LogFileHandler:
    class: Monolog\Handler\StreamHandler
    constructor:
      - "/var/www/silverstripe.log"
      - "notice"
    properties:
      Formatter: '%$Monolog\Formatter\HtmlFormatter'
      ContentType: text/html

  # Handler for displaying errors in the browser or CLI
  DisplayErrorHandler:
    class: SilverStripe\Logging\HTTPOutputHandler
    constructor:
      - "error"
    properties:
      Formatter: '%$SilverStripe\Logging\DebugViewFriendlyErrorFormatter'

  # Configuration for the "friendly" error formatter
  SilverStripe\Logging\DebugViewFriendlyErrorFormatter:
    class: SilverStripe\Logging\DebugViewFriendlyErrorFormatter
    properties:
      Title: "There has been an error"
      Body: "The website server has not been able to respond to your request"
```

[info]
In addition to Silverstripe CMS integrated logging, it is advisable to fall back to PHP's native logging functionality. A
script might terminate before it reaches the Silverstripe CMS error handling, for example in the case of a fatal error. Make
sure `log_errors` and `error_log` in your PHP ini file are configured.
[/info]

## Replacing default implementations

For most application, Monolog and its default error handler should be fine, as you can get a lot of flexibility simply
by changing that handlers that are used. However, some situations will call for replacing the default components with
others.

### Replacing the logger

Monolog comes by default with Silverstripe CMS, but you may use another PSR-3 compliant logger, if you wish. To do this,
set the `SilverStripe\Core\Injector\Injector.Monolog\Logger` configuration parameter, providing a new injector
definition. For example:

```yml
SilverStripe\Core\Injector\Injector:
  SilverStripe\Logging\ErrorHandler:
    class: Logging\Logger
    constructor:
     - 'alternative-logger'
```

If you do this, you will need to supply your own handlers, and the `Logger.handlers` configuration parameter will
be ignored.

### Replacing the error handler

The Injector service `SilverStripe\Logging\ErrorHandler` is responsible for initialising the error handler. By default
it:

- Create a `SilverStripe\Logging\MonologErrorHandler` object.
- Attach the registered service `Psr\Log\LoggerInterface` to it, to start the error handler.

`Core.php` will call `start()` on this method, to start the error handler.

This error handler is flexible enough to work with any PSR-3 logging implementation, but sometimes you will want to use
another. To replace this, you should registered a new service, `ErrorHandlerLoader`.  For example:

```yml
SilverStripe\Core\Injector\Injector:
  SilverStripe\Logging\ErrorHandler:
    class: MyApp\CustomErrorHandlerLoader
```

You should register something with a `start()` method.

## Filtering sensitive arguments

Depending on your PHP settings, error stacktraces may include arguments passed into functions. This could include sensitive
information such as passwords or API keys that you do not want leaking into your logs. The [Backtrace](api:SilverStripe\Dev\Backtrace)
class is responsible for rendering this backtrace and has a configuration variable `ignore_function_args` which holds the
names of functions for which arguments should be filtered. For functions in this list, the arguments are replaced with the
string `"<filtered>"`.

You can add either functions or class methods to this list - for functions just add them as a string. For class methods,
add an array which contains the fully namespaced class name and the name of the method. If the method is declared on an
interface, or on a class which is subclassed by other classes, just put the name of the interface or the superclass and
`Backtrace` will automatically filter out the classes which implement the interface or are subclasses of your superclass.

```yml
SilverStripe\Dev\Backtrace:
  ignore_function_args:
    - 'some_php_function'
    - ['App\MyClass', 'someMethod']
```

You should include any functions or methods here which have arguments that may be sensitive. If you are the author of a
module that other developers may use, it is best practice to include this configuration in the module. Developers should
not be expected to scan every Silverstripe module they use and add those declarations in their project configuration.

## Related lessons

- [Advanced environment configuration](https://www.silverstripe.org/learn/lessons/v4/advanced-environment-configuration-1)
