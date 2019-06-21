---
layout: post
title:  "Logging in Zend Framework Using the Event Manager"
date:   2019-06-21 11:00:00 +0000
categories: [zend]
---

Last year I was working on a refresh of our client control panel. The client control panel
is a web application that allows our users to gain an insight into their various services and
campaigns with us. The aim of the project was to create a brand new control panel using Zend 
Framework 3 components, as well as updating to the latest PHP version and using Material Design 
Components for the front end.

In our previous version of the control panel, there was very little infrastructure
in place for logging. Usage of logging was inconsistent and various different logging
libraries were used, depending on which developer had worked on that section of the site.

After some research, we narrowed it down to three potential options:

1. Use a `Logger` class with static methods for each of the potential
log levels. We had used this approach in several smaller applications but felt that it 
would not suit a large project such as this one. While this approach is easy to use and create, 
we didn't want to pollute our codebase with hundreds of static calls that cannot be mocked.
 
2. Inject an instance of a logger implementing `LoggerInterface` into any
class in which we wanted to log information. At the time, this seemed like it added unnecessary
overhead to the construction of each object.

3. Use the Zend [event-manager](https://github.com/zendframework/zend-eventmanager).
Emit log events (`LogEvent`) and attach a `LogListener` to the event manager. The 
`LogListener` then logs the information from the event using an instance of `LoggerInterface`.

The third approach seemed easy to implement and to understand, so we decided to use it in our application.
This post will cover how we implemented this, as well as reviewing this approach now that we've
spent over a year using it in production.

## Creating the Log Event

This is our `LogEvent` below. The event will be emitted by the class wishing to log information and then
caught by a listener attached to our event manager. We define public constants that map to log levels from [PSR-3](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-3-logger-interface.md).
We could have used the PSR-3 levels directly, however we wanted to provide a layer of abstraction between 
our `LogEvent` and  whichever logging library we used. This step is optional as most people will use PSR-3
compliant loggers.

```php
declare(strict_types=1);

namespace Application\Event;

use Psr\Log\LogLevel;
use Zend\EventManager\Event;

class LogEvent extends Event
{
    public const DEBUG     = LogLevel::DEBUG;
    public const INFO      = LogLevel::INFO;
    public const NOTICE    = LogLevel::NOTICE;
    public const WARNING   = LogLevel::WARNING;
    public const ERROR     = LogLevel::ERROR;
    public const CRITICAL  = LogLevel::CRITICAL;
    public const ALERT     = LogLevel::ALERT;
    public const EMERGENCY = LogLevel::EMERGENCY;
}
```

## Creating and Registering the Listener

Now that we have our event, we need to create a listener and register it with the event
manager. Listeners are attached to an event manager which is then responsible for reacting
to events that are emitted in the application.

Our listener implements the `ListenerAggregateInterface` which requires two methods: `attach`
for attaching one or more listeners and `detach` for removing those listeners. We use the 
`ListenerAggregateTrait` to implement the `detach` method.

```php
declare(strict_types=1);

namespace Application\Listener;

use Application\Event\LogEvent;
use Application\Model\LoggerAwareInterface as LoggerAware;
use Psr\Log\LoggerInterface;
use Zend\EventManager\EventInterface;
use Zend\EventManager\EventManagerInterface;
use Zend\EventManager\ListenerAggregateInterface;
use Zend\EventManager\ListenerAggregateTrait;

class LogListener implements ListenerAggregateInterface
{
    use ListenerAggregateTrait;

    /**
     * @var LoggerInterface
     */
    private $logger;

    /**
     * @param LoggerInterface $logger
     */
    public function __construct(LoggerInterface $logger)
    {
        $this->logger = $logger;
    }

    /**
     * @inheritdoc
     */
    public function attach(EventManagerInterface $events, $priority = 1)
    {
        $sharedManager = $events->getSharedManager();
        $this->listeners[] = $sharedManager->attach(LoggerAware::class, LogEvent::DEBUG, [$this, 'addDebug'], $priority);
        $this->listeners[] = $sharedManager->attach(LoggerAware::class, LogEvent::INFO, [$this, 'addInfo'], $priority);
        $this->listeners[] = $sharedManager->attach(LoggerAware::class, LogEvent::NOTICE, [$this, 'addNotice'], $priority);
        $this->listeners[] = $sharedManager->attach(LoggerAware::class, LogEvent::WARNING, [$this, 'addWarning'], $priority);
        $this->listeners[] = $sharedManager->attach(LoggerAware::class, LogEvent::ERROR, [$this, 'addError'], $priority);
        $this->listeners[] = $sharedManager->attach(LoggerAware::class, LogEvent::CRITICAL, [$this, 'addCritical'], $priority);
        $this->listeners[] = $sharedManager->attach(LoggerAware::class, LogEvent::ALERT, [$this, 'addAlert'], $priority);
        $this->listeners[] = $sharedManager->attach(LoggerAware::class, LogEvent::EMERGENCY, [$this, 'addEmergency'], $priority);
    }

    private function addRecord(string $loggerLevel, EventInterface $event)
    {
        $logMessage   = $event->getTarget();
        $logContext   = $event->getParams();

        $this->logger->log($loggerLevel, $logMessage, $logContext);
    }

    public function addDebug(EventInterface $event)
    {
        return $this->addRecord(LogEvent::DEBUG, $event);
    }

    public function addInfo(EventInterface $event)
    {
        return $this->addRecord(LogEvent::INFO, $event);
    }


    // Methods must be added for each log level listed in the LogEvent
}
```

We specifically use the `SharedEventManager` as it can be used when we want to attach 
listeners without yet having an instance of the class that composes an `EventManager`. 
In our case, that would be any class that wishes to emit `LogEvent`. As you can imagine, 
attaching the `LogListener` to every single class emitting events would be unwieldy.

We attach a listener and define a method for every log level defined in our `LogEvent`.
Some of the methods have been omitted from the example above for the sake of readability.
This allows us to log the events at different levels depending on the constant specified 
by the emitting class.

For example:

```php
// This event will be logged as INFO due to the constant specified
$this->getEventManager()->trigger(LogEvent::INFO, 'Example log message', ['context']);

// Whereas this event will be logged as CRITICAL
$this->getEventManager()->trigger(LogEvent::CRITICAL, 'All connections down', ['context']);
```

## The LoggerAwareInterface

As we are using the `SharedEventManager`, we need to provide an identifier when attaching
a listener. The identifier is used to identify classes that may emit the `LogEvent` and 
should therefore be listened to. We use `LoggerAwareInterface` aliased to `LoggerAware` as our
identifier for the listener. It extends the `EventManagerAwareInterface` as in order to 
`trigger` the `LogEvent`, the emitting class must have access to an instance of the event manager.
The `EventManagerAwareInterface` requires two methods to be implemented: `setEventManager` and 
`getEventManager`.

Any classes retrieved through the Zend service manager that implement the 
`EventManagerAwareInterface` will automatically have the event manager set. This makes
emitting events very easy, but does have a certain 'magic' quality to it that can be
unclear for developers who have not worked with the event manager before.

```php
declare(strict_types=1);

namespace Application\Model;

use Zend\EventManager\EventManagerAwareInterface;

interface LoggerAwareInterface extends EventManagerAwareInterface
{

}
```

For example, the class `Foo` implements the `LoggerAwareInterface` and has a method
`bar`. When the `bar` method is called, a `LogEvent` is emitted in the event manager
and handled by any listeners attached to the `LogEvent`. We've implemented the
`setEventManager` and `getEventManager` methods ourselves. In the `setEventManager`
method we use the class name as well as any interfaces it implements as identifiers
for the events. You'll recall earlier that we used `LoggerAwareInterface` as our identifier.

```php
class Foo implements LoggerAwareInterface {
    private $events;
    
    public function bar(): void
    {
        $this->getEventManager()->trigger(LogEvent::INFO, 'Example log message', ['context']);
    }
    
    /**
     * @inheritDoc
     */
    public function setEventManager(EventManagerInterface $events)
    {
       $className = get_class($this);
    
       $nsPos = strpos($className, '\\') ?: 0;
       $events->setIdentifiers(array_merge(
           [
               __CLASS__,
               $className,
               substr($className, 0, $nsPos),
           ],
           array_values(class_implements($className))
       ));
    
       $this->events = $events;
    
       return $this;
    }
    
    /**
     * @inheritDoc
     */
    public function getEventManager()
    {
       if (!$this->events) {
           $this->setEventManager(new EventManager());
       }
    
       return $this->events;
    }
}
```

## EventManagerAwareTrait

Duplicating the `setEventManager` and `getEventManager` in every class we wish to log
from would be a maintenance nightmare, as well as increasing 'visual debt'. We separated
these methods into an `EventManagerAwareTrait` that can be used in any class that wishes
to emit events.

```php
declare(strict_types=1);

namespace Application\Model;

use Zend\EventManager\EventManager;
use Zend\EventManager\EventManagerInterface;

trait EventManagerAwareTrait
{
    private $events;

    /**
     * Set the event manager instance used by this context
     *
     * @param  EventManagerInterface $events
     * @return $this
     */
    public function setEventManager(EventManagerInterface $events)
    {
        ...
    }

    /**
     * Retrieve the event manager
     *
     * Lazy-loads an EventManager instance if none registered.
     *
     * @return EventManagerInterface
     */
    public function getEventManager()
    {
        ...
    }
}
```

This simplifies our example to this:

```php
use Application\Model\EventManagerAwareTrait;

class Foo implements LoggerAwareInterface {
    use EventManagerAwareTrait;
    
    public function bar(): void
    {
        $this->getEventManager()->trigger(LogEvent::INFO, 'Example log message', ['context']);
    }
}
```

One thing to note is that you will not need to use the `EventManagerAwareTrait` when 
emitting events from a Zend controller, as it already implements `setEventManager` and
`getEventManager`.

## Attaching the Listener on Application Bootstrap

Almost there. There's still one more thing to do before we can start logging. We need to 
ensure our `LogListener` is attached to the event manager during the bootstrap phase of the 
application. We can do this in the `Module.php` for the `Application` module.

```php
declare(strict_types=1);

namespace Application;

use Application\Listener\LogListener;
use Zend\EventManager\EventInterface;
use Zend\ModuleManager\Feature\BootstrapListenerInterface;
use Zend\ModuleManager\Feature\ConfigProviderInterface;

class Module implements ConfigProviderInterface, BootstrapListenerInterface
{
    /**
     * @inheritdoc
     */
    public function getConfig()
    {
        return include __DIR__ . '/../config/module.config.php';
    }

    /**
     * @inheritdoc
     */
    public function onBootstrap(EventInterface $e)
    {
        $serviceManager     = $e->getApplication()->getServiceManager();
        $eventManager       = $e->getApplication()->getEventManager();
        $sharedEventManager = $eventManager->getSharedManager();

        // Logging Listener
        $logListener = $serviceManager->get(LogListener::class);
        $logListener->attach($eventManager);
    }
}

```

Great! Now we can easily trigger `LogEvent` from any class by simply implementing
the `LoggerAwareInterface` and using the `EventManagerAwareTrait`.

## Event Based Logging: 1 Year On

The new control panel was a success and was a huge step up from the previous version, both
for developers and our users. The logging works perfectly and has given us greater visibility
into our application whilst providing a consistent experience to developers.

However, this approach has heavily coupled our application with Zend's event manager component.
If we were to refactor our application to use a different framework, it would take much longer 
due to the coupling. This issue would not be as prevalent with the other implementations 
I listed at the beginning of this post.

In the past year we also welcomed some new developers to the team. These developers were in
junior positions and had no experience with an application like our control panel before.
The event management system features a large amount of 'magic' behaviour that isn't clear to 
newer developers, especially ones that are unfamiliar with event-driven architecture. I 
think it's important that something as vital as logging is immediately debuggable by everyone
in the team, regardless of skill level.

If I were to start this project again, I would pass an instance of `LoggerInterface` 
into any class that requires it. This approach may be more 'manual' but is much more explicit
and does not tie us to any framework in particular. If you are already working in an event-driven
system, you may decide that coupling with the event manager component isn't an issue. It's about
using the right tool for the job.

If you have any questions or feedback for this article, you can email me at ethan@messagecloud.com.

