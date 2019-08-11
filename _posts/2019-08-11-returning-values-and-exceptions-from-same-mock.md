---
layout: post
title:  "PHPUnit: Returning values and throwing exceptions from the same mocked method"
date:   2019-08-11 12:00:00 +0000
categories: ["testing"]
---

Recently I was writing a test case where an initial call to a method would return a value and
a subsequent call would throw an exception. I assumed that it could be achieved by using a 
combination of `will`, `willReturn` and `willThrowException`.

```php
$mockFoo     = $this->createMock(Foo::class);
$mockFactory = $this->createMock(FooFactory::class);

$mockFactory
    ->expects($this->exactly(2))
    ->method('make')
    ->with('Foo', 'Bar')
    ->will($this->willReturn($mockFoo), $this->throwException(new BarException()));
```

PHPUnit only supports one parameter to `will` and as such the exception was never thrown.
I was surprised to find that PHPUnit has no documented solution for this issue and I instead
resorted to scouring StackOverflow. I found [this](https://stackoverflow.com/a/6286827) answer 
which put me on the right track.

Instead of creating the `InvokedCount` matcher and immediately passing it to `expects`, we
can create it earlier and assign it to a variable. We can then replace our `will` method with
`willReturnCallback`, using `$matcher` in the callback. `InvokedCount` provides the method 
`getInvocationCount` which allows us to track how many times the method `make` has been invoked.
Using our callback, we can then handle each invocation separately.

```php
$mockFoo     = $this->createMock(Foo::class);
$mockFactory = $this->createMock(FooFactory::class);
$matcher     = $this->exactly(2);

$mockFactory
    ->expects($matcher)
    ->method('make')
    ->with('Foo', 'Bar')
    ->willReturnCallback(function () use ($matcher, $mockFoo) {
        if ($matcher->getInvocationCount() === 1) {
            return $mockFoo;
        }
    
        throw new BarException();
    });
```

It's worth noting that this solution is not a replacement for `willReturnOnConsecutiveCalls` and 
is only for use when your mocked method need to do more than return values.