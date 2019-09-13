---
published: false
title: "Advanced NestJS: How to build completely dynamic NestJS modules"
description: "tags: nestjs, nest, dependency injection, providers"
series:
canonical_url:
---

*John is a member of the NestJS core team*

So you've noticed the cool ways in which you can dynamically configure some of the out-of-the-box NestJS modules like `@nestjs/jwt` and `@nestjs/typeorm` and want to know how to do that in your own modules?  This is the right article for you! :smile:

### Intro

Over on the [NestJS documentation site](https://docs.nestjs.com), we recently added a new chapter on [dynamic modules](https://docs.nestjs.com/fundamentals/dynamic-modules). This is a fairly advanced chapter, and along with some recent significant improvements to the [asynchronous providers chapter](https://docs.nestjs.com/fundamentals/async-providers), Nest developers have some great new resources to help build configurable modules that can be assembled into complex, robust applications.

This article builds on that foundation and takes it one step further.  One of the hallmarks of NestJS is making **asynchronous programming** very straightforward.  Nest fully embraces Node.js Promises and the `async/await` paradigm. Coupled with Nest's signature Dependency Injection features, this is an extremely powerful combination. Let's look at how these features can be used to create *dynamically configurable modules*.  Mastering this skill lets you build modules that are re-usable in any context.  This enables fully context-aware re-usable packages (libraries), and lets you *assemble* apps that deploy smoothly across cloud providers and throughout the DevOps spectrum - from development to staging to production.

### Basic Dynamic Modules

In the [dynamic modules chapter](), the end result of the code sample is the ability to pass in options to configure a module while it is being imported.  After reading that chapter, we know how the following code snippet works via the dynamic module API.

```typescript
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { ConfigModule } from './config/config.module';

@Module({
  imports: [ConfigModule.register({ folder: './config' })],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

If you've played around with any NestJS modules - such as `@nestjs/typeorm` or `@nestjs/jwt` - you'll have noticed that they go beyond the features described in that chapter.  In addition to supporting the `register(...)` method shown above, they also support a fully **dynamic and asynchronous** version of the method.  For example, with the `@nestjs/jwt` module, you can use a construct like this:

```typescript
@Module({
  imports: [
    JwtModule.registerAsync({ useClass: ConfigService }),
  ]
})
```

With this construct, not only is the module dynamically configured, but the *options passed to the dynamic module are themselves constructed dynamically*. This is higher order functionality at its best. Configuration options are provided based on values *extracted from the environment* by the `ConfigService`, meaning they can be changed completely external to your feature code.  Compare that to hardcoding a parameter, as with `ConfigModule.register({ folder: './config'})`, and you can immediately see the win.

In this article, we'll further explore why you may need this feature, and how to build it. Make sure you have a firm grasp on the concepts in the [Custom providers](https://docs.nestjs.com/fundamentals/custom-providers) and [Dynamic modules](https://docs.nestjs.com/fundamentals/dynamic-modules) chapters before moving on to the next section.

### Async Options Providers Use-case

The section heading above is quite a mouthful!  What the heck is an *async options provider*?

To answer that, first consider once again that our example above (the `ConfigModule.register({ folder: './config'})` part) is passing a **static options object** in to the `register()` method.  As we learned in the *Dynamic modules* chapter, this options object is used to customize the behavior of the module.  (Review [that chapter](https://docs.nestjs.com/fundamentals/dynamic-modules) before proceeding if this concept is not familiar).  As mentioned, we're now going to take that concept *one step further*, and let our **options** object be provided dynamically at run-time.

### Dynamic Options Example

To give us a concrete working example for the remainder of this article, I'm going to introduce a new module. We'll then walk through how to use async options providers in that context.

I recently published the [@nestjsplus/massive package](https://github.com/nestjsplus/massive) to enable a Nest project to easily make use of the awesome [MassiveJS](https://massivejs.org/) library to do database work. We'll study the architecture of that package and use parts of it for the code we analyze in this article. Briefly, the package *wraps* the MassiveJS library into a Nest module.  MassiveJS provides its entire API to each consumer module (e.g., each feature module) through a `db` object that has methods like:

- `db.find()` to retrieve database records
- `db.update()` to update database records
- `db.myFunction()` to execute scripts or database procedures/functions

The primary function of the `@nestjsplus/massive` package is to make a connection to the database, and return the `db` object.  In our feature modules, we then access the database using methods hung off the `db` object, like those shown above. Right off the bat, it should be clear that in order to establish a database connection, we need to pass in some connection parameters.  In our case, with PostgreSQL, those parameters would be something like:

```js
{
  user: 'john',
  password: 'password',
  host: 'localhost',
  port: 5432,
  database: 'nest',
}
```

What we quickly realize is that it's not optimal to hard-code those connection parameters in our Nest app.  The conventional solution is to supply them through some sort of *Configuration Module*.  And that's exactly what we can do with Nest's `@nestjs/jwt` module as shown in the example above. That, in a nutshell, is the purpose of *async options providers*. Now let's figure out how to do that in our Massive module.

### Coding for Async Options Providers

To begin with, we can imagine supporting a module import statement using the same construct we found in `@nestjs/jwt`, like this:

```typescript
@Module({
  imports: [
    MassiveModule.registerAsync({ useClass: ConfigService })
  ],
})
```

If this doesn't look familiar, take a quick look at [this section](https://docs.nestjs.com/fundamentals/custom-providers#class-providers-useclass) of the *Custom providers* chapter. The similarities are deliberate. We're taking inspiration from the concepts learned from *Custom providers* to build a more flexible means of supplying options to our dynamic modules.

Let's dig in to the implementation. This is going to be a **long** section, so take a deep breath, maybe refill your coffee cup, but don't worry - we can do this! :smile:.  Bear in mind that we are walking through a design pattern that, once you understand it, you can **confidently** cut-and-paste as boilerplate to start building any module that needs dynamic configuration.  But before the cutting and pasting starts, let's make sure to understand the template so we can customize it to our needs. Just remember that you won't have to write this from scratch each time!

First, let's revisit what we expect to accomplish with the `registerAsync(...)` construct above.  Basically, we're saying "Hey module! I don't want to give you the option property values in code.  How about instead I give you a class that has a *method* you can call to get the option values?".  This would give us a great deal of flexibility in how we can generate the options dynamically at runtime.

This implies that our dynamic module is going to need to do a little more work in this case, as compared to the static options technique, to acquire its connection options.  We're going to work our way up to that result. We begin by formalizing some definitions.  We're trying to supply MassiveJS with its expected *connection options*, so first  we'll create an interface to model that:

```typescript
export interface MassiveConnectOptions {
  /**
   * server name or IP address
   */
  host: string;
  /**
   * server port number
   */
  port: number;
  /**
   * database name
   */
  database: string;
  /**
   * user name
   */
  user: string;
  /**
   * user password, or a function that returns one
   */
  password: string;
  /**
   * use SSL (it also can be a TSSLConfig-like object)
   */

  ...
}
```

There are actually more options available (we can see what they are from examining the [MassiveJS connection options](https://massivejs.org/docs/connecting) documentation), but let's keep the choices basic for now.  The ones we modelled are **required** to establish a connection.  As a side note, we're using JSDoc to document them so that we get a nice Intellisense developer experience when we later use the module.

The next concept to grapple with is as follows. Since our consumer module (the one **calling** `registerAsync()` to import the MassiveJS module) is handing us a class and expecting us to call a method on that class, we can surmise that we must be using a sort of factory pattern.  In other words, somewhere, we're going to have to instantiate that class, call a method on it, and use the result returned from that method call as our connection options, right?  Sounds (kind of) like a factory.  Let's go with that concept for now.

Let's describe our prospective factory with an interface.  The method could be something like `createMassiveConnectOptions()`.  It needs to return an object of type `MassiveConnectOptions` (the interface we defined a minute ago). So we have:

```typescript
interface MassiveOptionsFactory {
  createMassiveConnectOptions():
    Promise<MassiveConnectOptions> | MassiveConnectOptions;
}
```

Nice!  We can return the options object directly, or return a Promise that will resolve to an options object. Nest makes it super easy to support either.  Hence, we now see the "async" part of our *async options provider* coming into play.

Now, let's ask the following: what mechanism is going to actually *call* our factory function at run time, take the results, and make them available to the part of our code that needs them?  Hmmm... if only we had some general purpose mechanism.  Maybe a feature that we could register an arbitrary object with at run-time, and then have that object passed into a constructor.  Anybody got any ideas?  :wink:

Well of course, we've got the awesome NestJS Dependency Injection system at our disposal. That seems like a good fit! Let's figure out how to do that.

The *recipe* for binding something to the Nest IoC container, and later having it injected, is captured in an object called a *provider*. Let's whip up an *options provider* that will do just that. If you need a quick refresher course on custom providers, go ahead and re-read the [Custom providers chapter](https://docs.nestjs.com/fundamentals/custom-providers) now. It won't take long. I'll wait right here.

OK, so now you remember that we can define our *options provider* with a construct like the following. We already have an intuition that we need a factory provider, so this seems like the right construct:

```js
{
  provide: 'MASSIVE_CONNECT_OPTIONS',
  useFactory: // <-- we need to get our options factory inserted here!
  inject:     // <-- we need to supply injectable parameters for useFactory here
}
```

Let's try to tie a few things together. We're in pretty deep, so now's a good time to do a quick refresher on the big picture, and assess where we're at:

1. We're writing code that constructs, then returns, a dynamic module (our `registerAsync()` static method will house that code).
2. The dynamic module it returns can be imported into other feature modules, and provides a service (the thing that connects to the database and returns a `db` object).
3. That service needs to be *configured* at the time the module is constructed.  A better way to say this is that the service *depends on* a dynamically constructed *options object*.
4. We're going to construct that configuration options object at runtime, using a class that the consuming module hands us.
5. That class contains a method that knows how to supply an appropriate options object.
6. We're going to use the Nest Dependency Injection system to do the heavy lifting to manage that options object dependency for us.

OK, so we're working on steps 4, 5, and 6 right now.  We're not yet ready to assemble the entire dynamic module. Before we do that, we have to work out the mechanics of our *options provider*. Returning to that task, we should be able to see how to fill in the blanks in the skeleton *options provider* we sketched out earlier (see the lines annotated `<-- we need to...` above). We can fill in those values based on how the `registerAsync()` call was made:

```typescript
@Module({
  imports: [
    MassiveModule.registerAsync({ useClass: ConfigService })
  ],
})
```

Let's go ahead and fill them in now, as if we were working with a static object, just to see what we're trying to provide:

```typescript
{
  provide: 'MASSIVE_CONNECT_OPTIONS',
  useFactory: async (ConfigService) =>
    await ConfigService.createMassiveConnectOptions(),
  inject: [ConfigService]
}
```

So we've now figured out what our *options provider* should look like. Good so far? It's important to remember that the `'MASSIVE_CONNECT_OPTIONS'` provider is just fulfilling a dependency *inside the dynamic module*.  Now that I mention it, we haven't really looked at the service that depends on the `'MASSIVE_CONNECT_OPTIONS'` provider we're working so hard to supply. Let's take a quick moment to consider our ultimate objective - providing the service that connects to the database using this fancy *options provider*.  That service is, predictably enough, the `MassiveService` class.  It's surprisingly straightforward:

```typescript
@Injectable()
export class MassiveService {
  private _massiveClient;

  constructor(
    @Inject('MASSIVE_CONNECT_OPTIONS') private _massiveConnectOptions
  ) {}

  async connect(): Promise<any> {
    return this._massiveClient
      ? this._massiveClient
      : (this._massiveClient = await massive(this._massiveConnectOptions));
  }
}
```

The `MassiveService` class injects the connection options provider and uses that information to make the API call needed to create a database connection (the line: `massive(this._massiveConnectOptions)`). Once made, it caches the connection so it can return an existing connection on subsequent calls. That's it.  That's why we're jumping through hoops to be able to pass in our *options provider*.

We've now worked out the concepts and sketched out the piece parts of our dynamically configurable module.  We're ready to start assembling them.  First we'll write some *glue code* to pull this all together.  As we learned in the [dynamic modules](https://docs.nestjs.com/fundamentals/dynamic-modules) chapter, all that glue should live in the module definition class.  Let's create the `MassiveModule` class for that purpose.  We'll describe what's happening in this code just below it.

```typescript
@Global()
@Module({})
export class MassiveModule {

  /**
   *  public static register( ... )
   *  omitted here for brevity
   */

  public static registerAsync(
    connectOptions: MassiveConnectAsyncOptions,
  ): DynamicModule {
    return {
      module: MassiveModule,
      providers: [MassiveService].concat(
        this.createConnectProviders(connectOptions))
      ],
      exports: [MassiveService],
    };
  }

  private static createConnectProviders(
    options: MassiveConnectAsyncOptions,
  ): Provider[] {
    return [
      {
        provide: 'MASSIVE_CONNECT_OPTIONS',
        useFactory: async (optionsFactory: MassiveOptionsFactory) =>
          await optionsFactory.createMassiveConnectOptions(),
        inject: [options.useClass],
      },
      {
        provide: options.useClass,
        useClass: useClass,
      }
    ];
  }
```

Let's get a firm handle on what this code does. This is really where the rubber meets the road, so take time to understand it carefully. Consider that if we were to insert a logging statement displaying the return value from the following call:

```typescript
registerAsync({ useClass: ConfigService })
```

we'd see an object that looks pretty much like this:

```typescript
{
  module: MassiveModule,
  providers: [
    MassiveService,
    {
      provide: 'MASSIVE_CONNECT_OPTIONS',
      useFactory: async (optionsFactory: MassiveOptionsFactory) =>
        await optionsFactory.createMassiveConnectOptions(),
      inject: [ ConfigService ],
    },
    {
      provide: ConfigService,
      useClass: ConfigService,
    }
  ],
  exports: [ MassiveService ]
}
```

This should be pretty recognizable. To describe it in English, we're returning a dynamic module that declares **three** providers.

- The first provider is obviously the `MassiveService` itself, which we plan to use in our consumer's feature modules, so we re-export it.

- The second provider (`'MASSIVE_CONNECT_OPTIONS'`) is *only used internally* by the `MassiveService` to ingest the connection options it needs (notice that we do **not** re-export it).  Let's take a little closer look at that `useFactory` construct. Note that there's also an `inject` property, which is used to inject the `ConfigService` into the factory function.  This is described in detail [here in the Custom providers chapter](https://docs.nestjs.com/fundamentals/custom-providers#factory-providers-usefactory), but basically, the idea is that the factory function takes optional input arguments which, if specified, are resolved by injecting a provider from the `inject` property array.  You might be wondering where that `ConfigService` injectable comes from.  Read on :smile:.

- Finally, we have a third provider, also used only internally by our dynamic module (and hence not exported), which is our single private instance of the `ConfigService`.  So, Nest is going to instantiate a `ConfigService` inside the dynamic module context (this makes sense, right?  We told it to `useClass`, which means "create your own instance"), and that will be injected into the factory.

If you made it this far - congrats! That was the hardest part. We just worked out all the mechanics of assembling a dynamically configurable module.  The rest of the article is gravy!

One other thing that should be obvious from looking at the generated `useFactory` syntax above is that the `ConfigService` class must implement a `createMassiveConnectOptions()` method. This should be a familiar pattern if you're already using some sort of configuration module, but now you can see how it plugs into this construct.

### Variant Forms of Asynchronous Options Providers

What we've built so far allows us to configure the `MassiveModule` by handing it a class whose purpose is to dynamically provide connection options. Let's just remind ourselves again how that looks from the consumer perspective:

```typescript
@Module({
  imports: [
    MassiveModule.registerAsync({ useClass: ConfigService})
  ]
})
```

We can refer to this as configuring our dynamic module with a `useClass` technique (AKA a *class provider*).  Are there other options?  You may recall seeing several other similar patterns in the *Custom providers* chapter.  We can model our `registerAsync()` interface based on those patterns.  Let's sketch out what those techniques would look like from a consumer module perspective, and then we can easily add support for them.

#### Factory Providers: useFactory

While we ended up using a factory in the previous section, that was strictly *internal* to the dynamic module construction mechanics, not a part of the callable API.  What would `useFactory` look like when exposed as an option for our `registerAsync()` method?

```typescript
@Module({
  imports: [MassiveModule.registerAsync({
    useFactory: () => {
      return {
        host: "localhost",
        port: 5432,
        database: "nest",
        user: "john",
        password: "password"
      }
    }
  })]
})
```

In the sample above, we supplied a very simple factory *in place*, but we could of course plug in (or pass in a function implementing) any arbitrarily sophisticated factory as long as it returns an appropriate connections object.

#### Alias Providers: useExisting

This sometimes-overlooked construct is actually extremely useful.  In our context, it means we can ensure that we re-use an existing *options provider* rather than instantiating a new one.  For example, `useClass: ConfigService` will cause Nest to create and inject a new private instance of our `ConfigService`. In the real world, we'll probably want a single shared instance of the `ConfigService` injected anywhere it's needed, not a private copy.  The `useExisting` technique is our friend here.  Here's how it would look:


```typescript
@Module({
  imports: [MassiveModule.registerAsync({
    useExisting: ConfigService
  })]
})
```

### Supporting Multiple Async Options Providers Techniques

We're in the home stretch.  We're going to focus now on generalizing and optimizing our `registerAsync()` method to support the additional techniques described above.  When we're done, our module will support all three techniques:

1. useClass - to get a private instance of the options provider.
2. useFactory - to use a function as the options provider.
3. useExisting - to re-use an existing (shared, `SINGLETON`) service as the options provider.

I'm going to jump right to the code, as we're all getting weary now :wink:.  I'll describe the key elements below.

```typescript
@Global()
@Module({
  providers: [MassiveService],
  exports: [MassiveService],
})
export class MassiveModule {

  /**
   *  public static register( ... )
   *  omitted here for brevity
   */

  public static registerAsync(connectOptions: MassiveConnectAsyncOptions): DynamicModule {
    return {
      module: MassivconnectOptions.imports || [],eModule,
      imports:
      providers: [this.createConnectProviders(connectOptions)],
    };
  }

  private static createConnectProviders(
    options: MassiveConnectAsyncOptions,
  ): Provider[] {
    if (options.useExisting || options.useFactory) {
      return [this.createConnectOptionsProvider(options)];
    }

    // for useClass
    return [
      this.createConnectOptionsProvider(options),
      {
        provide: options.useClass,
        useClass: options.useClass,
      },
    ];
  }

  private static createConnectOptionsProvider(
    options: MassiveConnectAsyncOptions,
  ): Provider {
    if (options.useFactory) {

      // for useFactory
      return {
        provide: MASSIVE_CONNECT_OPTIONS,
        useFactory: options.useFactory,
        inject: options.inject || [],
      };
    }

    // For useExisting...
    return {
      provide: MASSIVE_CONNECT_OPTIONS,
      useFactory: async (optionsFactory: MassiveConnectOptionsFactory) =>
        await optionsFactory.createMassiveConnectOptions(),
      inject: [options.useExisting || options.useClass],
    };
  }
}
  ```

Before discussing the details of the code, let's cover a few superficial changes to make sure they don't trip you up.
- We now use the constant `MASSIVE_CONNECT_OPTIONS` in place of a string-valued token.  This is a simple best practice convention.
- Rather than listing `MassiveService` in the `providers` and `exports` properties of the dynamically constructed module, we promoted them up to live in the `@Module()` decorator metadata.  Why? Partly style, and partly to keep the code DRY.  The two approaches are equivalent.

### Fully Understanding the Code

You should be able to trace the path through this code to see how it handles each case uniquely.  I highly recommend you do the following exercise.  Construct an arbitrary `registerAsync()` registration call on paper, and walk through the code to predict what the returned dynamic module will look like.  This will strongly reinforce the patterns and help you firmly connect all the dots.

For example, if we were to code:

```typescript
@Module({
  imports: [MassiveModule.registerAsync({
    useExisting: ConfigService
  })]
})
```

We could expect a dynamic module to be constructed with the following properties:

```typescript
{
  module: MassiveModule,
  imports: [],
  providers: [
    {
      provide: MASSIVE_CONNECT_OPTIONS,
      useFactory: async (optionsFactory: MassiveConnectOptionsFactory) =>
        await optionsFactory.createMassiveConnectOptions(),
      inject: [ ConfigService ],
    },
  ],
}
```

(Note: because the module is preceded by an `@Module()` decorator that now lists `MassiveService` as a provider and exports it our resulting dynamic module will *also* have those properties.  Above we are just showing the elements that get added dynamically.)

Consider another question. How is the `ConfigService` available inside the factory for injection in this `useExisting` case? Well - heh heh - that's kind of a trick question.  In the sample above, I assumed that it was already visible inside the consuming module -- perhaps as a global module (one declared with `@Global()`).  Let's say that wasn't true, and that it lives in `ConfigModule` which has **not** somehow registered ConfigService as a global provider.  Can our code handle this?  Let's see.

Our registration would instead look like this:

```typescript
@Module({
  imports: [MassiveModule.registerAsync({
    useExisting: ConfigService,
    imports: [ ConfigModule ]
  })]
})
```

And our resulting dynamic module would look like this:

```typescript
{
  module: MassiveModule,
  imports: [ ConfigModule ],
  providers: [
    {
      provide: MASSIVE_CONNECT_OPTIONS,
      useFactory: async (optionsFactory: MassiveConnectOptionsFactory) =>
        await optionsFactory.createMassiveConnectOptions(),
      inject: [ ConfigService ],
    },
  ],
}
```

Do you see how the pieces fit together?

Another exercise is to ponder the difference in the code paths when you use `useClass` vs. `useExisting`.  The important point is how we either instantiate a `ConfigService` class, or inject an existing one.  It's worth working through those details, as the concepts will give you a full picture of how NestJS modules can fit together in a coherent way.  But this article is already too long, so I'll leave that as an exercise for you, dear reader. :smile:

If you have questions, feel free to ask in the comments below!

### Conclusion

The patterns illustrated above are used throughout Nest's add-on modules, like `@nestjs/jwt` and `@nestjs/typeorm`.  Hopefully you now see not only how powerful these patterns are, but how you can make use of them in your own project.

As a next step, you may want to consider browsing through the source code of those modules, now that you have a roadmap. You can also see a slightly evolved version of the code in this article in the [@nestjsplus/massive repository](https://github.com/nestjsplus/massive) (while you're there, maybe give it a quick :star: if you like this article :wink:). The main difference between the code in this article and that repo is that the production version needs to handle **multiple** asynchronous options providers, so there's a tiny bit more plumbing.

Now you can confidently start using these powerful patterns in your own code to create robust and flexible modules that work reliably in a wide variety of contexts.

As a final bonus, if you're building an Open Source package for public use, just combine this technique with the steps described in my last article [on publishing NPM packages](https://dev.to/nestjs/publishing-nestjs-packages-with-npm-21fm), and you're all set.

Feel free to ask questions, make comments or suggestions, or just say hello in the comments below.  And join us at [Discord](https://discord.gg/G7Qnnhy) for more happy discussions about NestJS.  I post there as *Y Prospect*.
