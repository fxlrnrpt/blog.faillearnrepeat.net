---
title: NodeJS logging made easy
summary: How many times did you write `logger.info('ServiceName.methodName.')` and `logger.info('ServiceName.methodName -> done.')` for each and every method of your service you wanted to log? Now it is automated!
date: 2019-04-07
authors:
  - admin
tags:
  - web
---

How many times did you write `logger.info('ServiceName.methodName.')` and `logger.info('ServiceName.methodName -> done.')` for each and every method of your service you wanted to log? Would you like it to be automated and has the same constant signature across your whole app? If that's so, we're very much alike, we have suffered the same pain too many times, and now we could finally try to resolve it. Together. Ladies and gentlemen, let me introduce... [class-logger](https://github.com/fxlrnrpt/class-logger)!

## "The why" of [class-logger](https://github.com/fxlrnrpt/class-logger)

Engineers are often perfectionists. Perfectionists to the extreme. We like neat abstractions. We like clean code. We see beauty in artificial languages other people cannot even read. We like manufacturing small digital universes, living by the rules we set. We like all of that, probably, because we're very lazy. No, we're not afraid of work, but we hate doing any work that can be automated.

Having written couple thousands of lines of logging code only, we usually come up with certain patterns, standardizing what we want to log. Yet we still have to apply those pattern manually. So the core idea of [class-logger](https://github.com/fxlrnrpt/class-logger) is to provide a declarative highly-configurable standardized way to log messages before and after the execution of a class method.

## Quick start

Let's hit the ground running and see what the actual code looks like.

```ts
import { LogClass, Log } from 'class-logger'

@LogClass()
class ServiceCats {
  @Log()
  eat(food: string) {
    return 'purr'
  }
}
```

This service is going to log three times:

- At its creation with a list of arguments passed to the constructor.
- Before `eat` is executed with a list of its arguments.
- After `eat` is executed with a list of its arguments and its result.

In words of code:

```ts
// Logs before the actual call to the constructor
// `ServiceCats.construct. Args: [].`
const serviceCats = new ServiceCats()

// Logs before the actual call to `eat`
// `ServiceCats.eat. Args: [milk].`
serviceCats.eat('milk')
// Logs after the actual call to `eat`
// `ServiceCats.eat -> done. Args: [milk]. Res: purr.`
```

[Live demo](https://stackblitz.com/edit/class-logger-demo-cats-quick-start).

What else could we log? Here's the complete list of events:

- Before class construction.
- Before synchronous and asynchronous static and non-static methods and functional properties.
- After synchronous and asynchronous static and non-static methods and functional properties.
- Errors of synchronous and asynchronous static and non-static methods and functional properties.

> Functional property is an arrow function assigned to a property (`class ServiceCats { private meow = () => null }`).

## Adjusting it to our needs

So far so good, but we've been promised "customizable", right? So how can we tweak it?

[class-logger](https://github.com/fxlrnrpt/class-logger) provides three layers of hierarchical config:

- Global
- Class
- Method

At every method call, all three of them are evaluated and merged together from top to bottom. There's some sane default global config, so you can use the library without any configuration at all.

### Global config

It's the app-wide config. Can be set with `setConfig` call.

```ts
import { setConfig } from 'class-logger'

setConfig({
  log: console.info,
})
```

### Class config

It has an effect over every method of your class. It could override the global config.

```ts
import { LogClass } from 'class-logger'

setConfig({
  log: console.info,
})

@LogClass({
  // It overrides global config for this service
  log: console.debug,
})
class ServiceCats {}
```

### Method config

It affects only the method itself. Overrides class config and, therefore, global config.

```ts
import { LogClass } from 'class-logger'

setConfig({
  log: console.info,
})

@LogClass({
  // It overrides global config for this service
  log: console.debug,
})
class ServiceCats {
  private energy = 100

  @Log({
    // It overrides class config for this method only
    log: console.warn,
  })
  eat(food: string) {
    return 'purr'
  }

  // This method stil uses `console.debug` provided by class config
  sleep() {
    this.energy += 100
  }
}
```

[Live demo](https://stackblitz.com/edit/class-logger-demo-hierarchical-config)

### Configuration options

Well, we've learned how to alter the defaults, yet it wouldn't hurt to cover what there is to configure, huh?

> [Here you can find lots of useful config overrides](https://github.com/fxlrnrpt/class-logger#examples).

> [Here's the link to the interface of the config object in case you speak TypeScript better than English :)](https://github.com/fxlrnrpt/class-logger#configuration-object)

Configuration object has these properties:

#### log

It's a function that does the actual logging of the final formatted message. It's used to log these events:

- Before class construction.
- Before synchronous and asynchronous static and non-static methods and functional properties.
- After synchronous and asynchronous static and non-static methods and functional properties.

Default: `console.log`

#### logError

It's a function that does the actual logging of the final formatted error message. It's used to log this one and only event:

- Errors of synchronous and asynchronous static and non-static methods and functional properties.

Default: `console.error`

#### formatter

It's an object with two methods: `start` and `end`. It formats logging data into the final string.

`start` formats messages for these events:

- Before class construction.
- Before synchronous and asynchronous static and non-static methods and functional properties.

`end` formats messages for these events:

- After synchronous and asynchronous static and non-static methods and functional properties.
- Errors of synchronous and asynchronous static and non-static methods and functional properties.

Default: `new ClassLoggerFormatterService()`

#### include

The configuration of what should be included in the message.

##### args

It could be either a boolean or an object.

If it's a boolean, it sets whether to include the list of arguments (remember that `Args: [milk]`?) into both, start (before construction and before method call) and end (after method call, error method call), messages.

If it's an object, it should have two boolean properties: `start` and `end`. `start` includes/excludes the list of arguments for start messages, `end` does the same for end messages.

Default: `true`

##### construct

A boolean flag setting whether to log class construction or not.

Default: `true`

##### result

Another boolean flag setting whether to include a return value of a method call or an error thrown by it. Remember `Res: purr`? If you set this flag to `false` there will be no `Res: purr`.

Default: `true`

##### classInstance

Once again, either a boolean or an object.
If you enable it, a stringified representation of your class instance will be added to the logs. In other words, if your class instance has some properties, they will be converted to a JSON string and added to the log message.

Not all properties will be added. [class-logger](https://github.com/fxlrnrpt/class-logger) follows this logic:

- Take own (non-prototype) properties of an instance.
  - Why? It's a rare case when your prototype changes dynamically, therefore it hardly makes any sense to log it.
- Drop any of them that have `function` type.
  - Why? Most of the time `function` properties are just immutable arrow functions used instead of regular class methods to preserve `this` context. It doesn't make much sense to bloat your logs with stringified bodies of those functions.
- Drop any of them that are not plain objects.
  - What objects are plain ones? `ClassLoggerFormatterService` considers an object a plain object if its prototype is strictly equal to `Object.prototype`.
  - Why? Often we include instances of other classes as properties (inject them as dependencies). Our logs would become extremely fat if we included stringified versions of these dependencies.
- Stringify what's left.

```ts
class ServiceA {}

@LogClass({
  include: {
    classInstance: true,
  },
})
class Test {
  private serviceA = new ServiceA()
  private prop1 = 42
  private prop2 = { test: 42 }
  private method1 = () => null

  @Log()
  public method2() {
    return 42
  }
}

// Logs to the console before the class' construction:
// 'Test.construct. Args: []. Class instance: {"prop1":42,"prop2":{"test":42}}.'
const test = new Test()

// Logs to the console before the method call:
// 'Test.method2. Args: []. Class instance: {"prop1":42,"prop2":{"test":42}}.'
test.method2()
// Logs to the console after the method call:
// 'Test.method2 -> done. Args: []. Class instance: {"prop1":42,"prop2":{"test":42}}. Res: 42.'
```

Default: `false`

## Taking control over formatting

So what if you like the overall idea, but would like your messages to look differently? You can take complete control over formatting passing your own custom formatter.

You could write your own formatter from scratch. Totally. Yet we're not going to cover this option here (if you're really interested in that, address the ["Formatting" section](https://github.com/fxlrnrpt/class-logger#formatting) of the README).

The fastest and, probably, easiest thing to do is to subclass a built-in default formatter - `ClassLoggerFormatterService`.

`ClassLoggerFormatterService` has these protected methods, serving as building blocks of the final message:

- `base`
  - Returns the class name with the method name. Example: `ServiceCats.eat`.
- `operation`
  - Returns `-> done` or `-> error` based on whether it was a successful execution of a method or an error one.
- `args`
  - Returns a stringified list of arguments. Example: `. Args: [milk]`. It uses [fast-safe-stringify](https://github.com/davidmarkclements/fast-safe-stringify) for objects under the hood.
- `classInstance`
  - Returns a stringified class instance. Example: `. Class instance: {"prop1":42,"prop2":{"test":42}}`. If you choose to include class instance, but it is not available (that's how it is for static methods and class construction), it returns `N/A`.
- `result`
  - Returns a stringified result of the execution (even if it was an error). Uses [fast-safe-stringify](https://github.com/davidmarkclements/fast-safe-stringify) to serialize objects. A stringified error will be composed of the following properties:
  - Name of the class (function) the error was created with (`error.constructor.name`).
  - Error code (`error.code`).
  - Error message (`error.message`).
  - Error name (`error.name`).
  - Stack trace (`error.stack`).
- `final`
  - Returns `.`. Just `.`.

The `start` message consists of:

- `base`
- `args`
- `classInstance`
- `final`

The `end` message consists of:

- `base`
- `operation`
- `args`
- `classInstance`
- `result`
- `final`

You can override any one of those building-block methods. Let's take a look at how we could add a timestamp. I'm not saying we should. [pino](https://github.com/pinojs/pino), [winston](https://github.com/winstonjs/winston) and many other loggers are capable of adding timestamps on their own. SO the example is purely educative.

```ts
import {
  ClassLoggerFormatterService,
  IClassLoggerFormatterStartData,
  setConfig,
} from 'class-logger'

class ClassLoggerTimestampFormatterService extends ClassLoggerFormatterService {
  protected base(data: IClassLoggerFormatterStartData) {
    const baseSuper = super.base(data)
    const timestamp = Date.now()
    const baseWithTimestamp = `${timestamp}:${baseSuper}`
    return baseWithTimestamp
  }
}

setConfig({
  formatter: new ClassLoggerTimestampFormatterService(),
})
```

[Live demo](https://stackblitz.com/edit/class-logger-demo-custom-formatter-add-timestamp)

## Conclusion

Please, don't forget to follow [installation steps](https://github.com/fxlrnrpt/class-logger#installation) and get acquainted with [requirements](https://github.com/fxlrnrpt/class-logger#requirements) before you decide to use this library.

Hopefully, you've found something useful for your project. Feel free to communicate your feedback to me! I most certainly appreciate any criticism and questions.