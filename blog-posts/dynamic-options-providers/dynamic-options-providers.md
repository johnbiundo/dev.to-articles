published: false
title: "Build completely dynamic NestJS modules"
description: "tags: nestjs, nest, dependency injection, providers"
series:
canonical_url:

*John is a member of the NestJS core team*

### Intro

Over on the [NestJS documentation site](https://docs.nestjs.com), we recently added a new chapter on [dynamic modules](https://docs.nestjs.com/fundamentals/dynamic-modules). This is a fairly advanced chapter, and along with some recent significant improvements to the [asynchronous providers chapter](https://docs.nestjs.com/fundamentals/asynchronous-providers), Nest developers have some great new resources to help build configurable modules that can be assembled into complex, robust applications.

This article builds on that foundation and takes it one step further.  One of the hallmarks of NestJS is making **asynchronous programming** very straightforward.  Nest fully embraces Promises and the `async/await` paradigm. Coupled with Nest's signature Dependency Injection features, this is an extremely powerful combination. Let's look at how these features can be used to create *dynamically configurable modules*.  Mastering this capability enables you to build modules that are re-usable in any context.  This enables fully context-aware re-usable packages (libraries), as well as apps that deploy smoothly across cloud providers and throughout the DevOps spectrum - from development to staging to production.

### Basic Dynamic Modules

In the [dynamic modules chapter](), the end result of the code sample is the ability to pass in options to configure a module while it is being imported.  At reading the chapter, we know how a construct like the following works through the dynamic module API.

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

If you've played around with any NestJS modules - such as `@nestjs/typeorm`, `@nestjs/jwt` or `@nestjs/passport` - you'll have noticed that they go beyond the features described in that chapter.  In addition to supporting the `register(...)` method shown above, they also support a fully **dynamic and asynchronous** version of the method.  For example, with the `@nestjs/jwt` module, you can use a construct like this:

```typescript
@Module({
  imports: [JwtModule.registerAsync({
    useClass: ConfigService,
    imports: [ConfigModule],
  })]
})
```

With this construct, not only is the module dynamically configured, but the **options** passed to the dynamic module are themselves constructed dynamically. This is higher order functionality at its best. Configuration options are provided based on values currently available from the `ConfigService`, meaning they can be changed completely external to the implementation.  Compare that to hardcoding a parameter, as with `ConfigModule.register({ folder: './config'})`, and you can immediately see the win.

In this article, we'll further explore why you may need this feature, and how to build it. Make sure you have a firm grasp on the concepts in the [Custom providers](https://docs.nestjs.com/fundamentals/custom-providers) and [Dynamic modules](https://docs.nestjs.com/fundamentals/dynamic-modules) chapters before moving on to the next section.

### Async Options Providers Use-case

That section heading is a mouthful!  What the heck is an *async options provider*?

To answer that, first consider that our example above (specifically, the `ConfigModule.register({ folder: './config'})` part) is passing a **static options object** in to the `register()` method.  As we learned in the *Dynamic modules* chapter, this options object is used to customize the behavior of the module.  (Review [that chapter]() before proceeding if this concept is not familiar).  As mentioned, we now want to take that concept *one step further*, and let our **options** object be provided dynamically at run-time.

To give us a concrete working example for the remainder of this article, I'm going to introduce a new module. We'll then walk through how to use dynamic options providers in that context.

I recently published the [@nestjsplus/massive package](https://github.com/nestjsplus/massive) to enable a Nest project to seamlessly integrate with the awesome [MassiveJS](https://massivejs.org/) library. We'll study the architecture of that package and use parts of it for the code we analyze in this article. Briefly, the package *wraps* the MassiveJS library into a Nest module.  MassiveJS provides its entire API to each consumer module (e.g., each feature module) through a `db` object that has methods like:

- `db.find()` to retrieve database records
- `db.update()` to update database records
- `db.myFunction()` to execute scripts or database procedures/functions

The primary function of the `@nestjsplus/massive` package is to make a connection to the database, and return the `db` object.  In our feature modules, we then access the database via methods hung off the `db` object, as shown above. Right off the bat, it should be clear that in order to establish a database connection, we need to pass in some connection parameters.  In our case, with PostgreSQL, those parameters would be something like:

```json
{
  user: 'john',
  password: 'password',
  host: 'localhost',
  port: 5432,
  database: 'nest',
}
```

What we quickly realize is that it's not optimal to hard-code those connection parameters in our Nest app.  The conventional solution is to supply them through some sort of *Configuration Module*.  And that's exactly what we're doing in the JWT example above. Now let's figure out how to do that in our Massive module.

To begin with, we can imagine modifying our module import using a construct like this:

```typescript
@Module({
  imports: [MassiveModule.registerAsync({
    useClass: ConfigService,
    imports: [ConfigModule],
  })]
})
```

What's happening here?  In the *static options* case, when we use a `register()` method, we pass in a static options object.  Instead, we'd like to have this `registerAsync()` method use an *options provider* that (someone? Perhaps the NestJS runtime system on our behalf?) can call upon to supply the object dynamically.

Let's look again at the object we passed to `registerAsync` above:

```json
{
  useClass: ConfigService,
  imports: [ConfigModule],
}
```

If this doesn't look familiar, take a quick look at this section of the [Custom providers chapter](https://docs.nestjs.com/fundamentals/custom-providers#class-providers-useclass). The similarities are deliberate. We're applying the lessons learned from *Custom providers* to build a more flexible means of supplying options to our dynamic modules - hence the name *async options provider*.

### Coding for Async Options Providers

So how do we implement this?  Let's dig in. This is going to be a **long** section, but we can do this! :)  Bear in mind that we are walking through a design pattern that, once you understand it, you can **confidently** cut-and-paste as boilerplate to start any sophisticated module that needs dynamic configuration.  As always, before the cutting and pasting starts, it's important to understand the template so you can customize it to your needs. Just remember that you won't have to write this from scratch each time!

Let's first discuss what we expect to accomplish with the `registerAsync(...)` construct above.  Basically, we're saying "Hey module! I don't want to give you the option property values in code.  How about instead I give you a class that has a *method* you can call to get the option values?".  This would give us a great deal of flexibility in how we can generate the options dynamically at runtime.

This implies that our dynamic module is going to need to do a little more work in this case, as compared to the static options technique, to acquire its connection options.  We're going to work up to that. We begin by formalizing some definitions.  We're trying to supply MassiveJS with its expected *connection options*, so first  we'll create an interface to model that:

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

There are actually more options available (we know what they are from examining the [MassiveJS connection options](https://massivejs.org/docs/connecting)) documentation, but let's keep the choices basic for now.  The ones we modelled are **required** to establish a connection.  As a side note, we're using JSDoc to document them so that we get a nice Intellisense developer experience when we later use the module.

The next concept to grapple with is as follows. Since our consumer module (the one **calling** `registerAsync()`, i.e., the one we're importing our dynamic module **into**) is handing us a class and expecting us to call a method on that class, we can deduce that we must be using a sort of factory pattern.  In other words, somewhere, we're going to have to instantiate that class, call a method on it, and use the result returned from that method call as our connection options, right?  Sounds (kind of) like a factory.  Let's go with that concept for now.

Let's describe that factory with an interface.  The method could be something like `createMassiveConnectOptions()`.  It's going to return an object of type `MassiveConnectOptions` (the interface we defined a minute ago). So we have:

```typescript
interface MassiveOptionsFactory {
  createMassiveConnectOptions():
    Promise<MassiveConnectOptions> | MassiveConnectOptions;
}
```

Now what mechanism is going to actually *call* our factory function at run time, take the results, and make them available to the part of our code that needs them?  Hmmm... if only we had some general purpose mechanism.  Maybe a feature that we could register an arbitrary object with at run-time, and then have that object passed into a constructor.  Anybody got any ideas?  ;)

Well of course, we've got the awesome NestJS Dependency Injection system at our disposal. That seems like a good fit!

The *recipe* for binding something to the Nest IoC container, and later having it injected, is captured in an object called a *provider*. Let's whip up an *options provider* that will do just that. If you need a quick refresher course on custom providers, go ahead and re-read the [Custom providers chapter](https://docs.nestjs.com/fundamentals/custom-providers) now. It won't take long. I'll wait right here.

OK, so now you remember that we can define our *options provider* with a construct like this:

```json
{
  provide: 'MASSIVE_CONNECT_OPTIONS',
  useFactory: // <-- we need to get our options factory inserted here!
  imports:    // <-- we need to import a module here if our factory lives in a separate one
}
```

Let's tie a few things together.  Recall where we started (with our consumer module doing a dynamic import via `registerAsync()`), and look again at the object passed in to `registerAsync()`:

```typescript
@Module({
  imports: [MassiveModule.registerAsync({
    useClass: ConfigService,
    imports: [ConfigModule],
  })]
})
```

With those points covered, we can now start to analyze our `registerAsync()` static method.  Ultimately, it needs to return a dynamic module.  We're in pretty deep, so now's a good time to do a quick refresher on the big picture, and assess where we're at:

1. We're writing code that constructs a dynamic module (our `registerAsync()` static method will house that code).
2. The dynamic module provides a service (the thing that connects to the database and returns a `db` object) that can be imported into other feature modules.
3. That service needs to be configured.  A better way to say this is that the service *depends on* a dynamically constructed *options object*.
4. We're going to construct that options object at runtime by using a class that the consuming module hands us.
5. That class contains a method that knows how to supply an appropriate options object.
6. We're going to use the Nest Dependency Injection system to do the heavy lifting to manage that options object for us.

OK, so we're working on steps 4, 5, and 6 right now.  We're not yet ready to assemble the entire dynamic module. Before we do that, we have to work out the mechanics of our *options provider*. Returning to that task, we can now see how to fill in the blanks in the skeleton *options provider* we sketched out earlier (see the lines annotated `<-- we need...` above). Let's just fill in those values as if we had a static object to see what we're trying to provide:

```typescript
{
  provide: 'MASSIVE_CONNECT_OPTIONS',
  imports: [ConfigModule],
  useFactory: async () =>
    await ConfigService.createMassiveConnectOptions(),
  inject: [ConfigService]
}
```

So we've now figured out what our *options provider* should look like. Good so far?

Our next step is to do the assembly of the dynamic module.  Remember that the *options provider* is, after all, just fulfilling a dependency in that module.  Now that we mention it, we haven't really looked at that service that depends on the `'MASSIVE_CONNECTIONS_OBJECT'`. let's take a very short detour to consider our ultimate objective - providing the service that needs this fancy *options provider*.  That is, of course, a `MassiveService` class.  It's surprisingly straightforward:

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

The `MassiveService` class injects the connection options provider and uses that information to make the API call needed to make a database connection: `massive(this_massiveConnectOptions)`. Once made, it caches the connection so it can provide an existing connection on subsequent calls. That's it.  That's why we're jumping through hoops to be able to pass in our *options provider*.

We've now worked out the concepts and sketched out the piece parts of our dynamically configurable module.  Let's start assembling them.  First we'll write some *glue code* to pull this all together.  As we learned in the [dynamic modules](https://docs.nestjs.com/fundamentals/dynamic-modules) chapter, all that glue should live in the module definition class.  Let's create the `MassiveModule` class for that purpose.  We'll describe what's happening in the this code just below it.

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
      imports: connectOptions.imports || [],
      providers: [
        this.createConnectProvider(connectOptions),
        MassiveService,
      ],
      exports: [MassiveService],
    };
  }

  private static createConnectProvider(
    options: MassiveConnectAsyncOptions,
  ): Provider {
    return {
      provide: 'MASSIVE_CONNECT_OPTIONS',
      useFactory: async (optionsFactory: MassiveOptionsFactory) =>
        await optionsFactory.createMassiveConnectOptions(),
      inject: [options.useClass],
    };
  }
```

If we were to insert a logging statement displaying the return value from `registerAsync()`, we'd see an object that looks pretty much like this:

```typescript
{
  module: MassiveModule,
  imports: [],
  providers: [
    {
      provide: 'MASSIVE_CONNECT_OPTIONS',
      useFactory: async (optionsFactory: MassiveOptionsFactory) =>
        await optionsFactory.createMassiveConnectOptions(),
      inject: [ ConfigService ],
    },
    MassiveService
  ],
  exports: [ MassiveService ]
}
```

This should be pretty recognizable. To describe it in English, we're returning a dynamic module that declares two providers.  One of the providers is obviously the `MassiveService` itself that we'll be using in our consumer modules, and so we re-export it.  The other provider is *only used internally* by the `MassiveService` to ingest the connection options it needs.  Let's take a little closer look at that factory provider. Note that it has an `inject` property to inject the `ConfigService` into the factory function.  This is described in detail [here in the Custom providers chapter](https://docs.nestjs.com/fundamentals/custom-providers#factory-providers-usefactory), but basically, the idea is that the factory function takes optional input arguments which, if specified, are resolved by injecting a provider from the `inject` property array.

One other thing that should be obvious from looking at the generated `useFactory` syntax above is that the `ConfigService` class must implement a `createMassiveConnectOptions()` method. This should be a familiar pattern if you're already using some sort of configuration module, but now you can see how it plugs into this construct. Later, when we finalize our code, we'll make sure to use types to ensure that TypeScript, and our development tooling, assists us in making sure we supply a conforming class in `useFactory`.

### Variant Forms of Asynchronous Options Providers

What we've built so far allows us to configure the `MassiveModule` by handing it a class whose purpose is to dynamically provide connection options. Let's just remind ourselves again how that looks from the consumer perspective:

```typescript
@Module({
  imports: [MassiveModule.registerAsync({
    useClass: ConfigService,
    imports: [ConfigModule],
  })]
})
```

We can refer to this as configuring our dynamic module with a `useClass` technique (AKA a *class provider*).  Are there other options?  You may recall seeing several other similar patterns in the *custom providers* chapter.  We can model our `registerAsync()` interface based on those patterns.  Let's sketch out what those techniques would look like from a consumer module perspective, and then we can easily add support for them.

#### Factory Providers: useFactory

While we mentioned using a factory in the previous section, that was *internal* to the module mechanics.  What would `useFactory` look like in terms of our `registerAsync()` method?

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

In the sample above, we supplied a very simple factory *in place*, but we could of course plug in any arbitrarily sophisticated factory as long as it returns an appropriate connections object.

#### Alias Providers: useExisting

This sometimes overlooked construct is actually extremely useful.  In our context, it means we can ensure that we re-use an existing provider rather than instantiating a new one.  For example, `useClass: ConfigService` will cause Nest to create and inject a new private instance of our `ConfigService`. In the real world, we'll probably want one shared instance of the `ConfigService` injected anywhere it's needed, not a private copy.  The `useExisting` technique is our friend here.  Here's how it would look:


```typescript
@Module({
  imports: [MassiveModule.registerAsync({
    useExisting: ConfigService
  })]
})
```

### Supporting Multiple Async Options Providers Techniques

We're in the home stretch.  We're going to focus now on generalizing and abstracting our `registerAsync()` method to support the additional methods described above.  I'm going to jump right to the code, as we're all getting weary now ;).  I'll describe the key elements below.

```typescript
@Global()
@Module({
  providers: [MassiveService, connectionFactory],
  exports: [MassiveService, connectionFactory],
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

Before discussing the details of the code, let's cover a few superficiaal changes to the prior implementation to make sure we don't trip you up.  These changes are mostly just an artifact of avoiding a bit of unnecessary complexity in the earlier example.
- Yes, we did change the name (from singular to plural) and return type (to array) of the `createConnectProvider` method.
- We use the constant `MASSIVE_CONNECT_OPTIONS` in place of the string-valued token above.
- Rather than listing `MassiveService` in the `providers` and `exports` properties of the dynamically constructed module, we promoted them up to the decorator.  Why? Partly style, and partly to keep the code DRY.  The two approaches are equivalent.

You should be able to trace the path through this code to see how it handles each case uniquely.  I highly recommend you do the following.  Construct an arbitrary `registerAsync()` registration call, and walk through the code to predict what the returned dynamic module will look like.  This will strongly reinforce the patterns and help you firmly connect all the dots.

For example, if we were to write:

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

And of course, because the module is preceded by an `@Module()` decorator listing `MassiveService` as a provider and exporting it, our resulting dynamic module will have those properties.

Consider another question. How is the `ConfigService` available inside the factory for injection? Kind of a trick question.  In the sample above, I assumed that it was already visible inside the consuming module -- perhaps as a global module.  Let's say that wasn't true, and that it lives in `ConfigModule`.  Can our code handle this?  Let's see.

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
  imports: [],
  providers: [
    {
      imports: [ ConfigModule ]
      provide: MASSIVE_CONNECT_OPTIONS,
      useFactory: async (optionsFactory: MassiveConnectOptionsFactory) =>
        await optionsFactory.createMassiveConnectOptions(),
      inject: [ ConfigService ],
    },
  ],
}
```

Do you see how the pieces fit together?  If you have questions, feel free to ask in the comments below!

The patterns illustrated above are used throughout Nest's add-on modules, like `@nestjs/jwt` and `@nestjs/typeorm`.  You can now confidently use them in your own code to create robust and flexible modules that work reliably in a wide variety of contexts.

Feel free to ask questions, make comments or suggestions, or just say hello in the comments below.  And join us at [Discord](https://discord.gg/G7Qnnhy) for more happy discussions about NestJS.  I post there as *Y Prospect*.
