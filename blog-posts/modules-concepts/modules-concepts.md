---
published: false
title: "Understanding NestJS Modules"
description: "tags: nestjs, nest, dependency injection, providers"
series:
canonical_url:
---

_John is a member of the NestJS core team_

Modules are basic building blocks of NestJS applications. Before you know it, you're plunged right into using them. The [NestJS docs](https://docs.nestjs.com) cover the features of modules and _how_ to use them comprehensively. What may be missing for people new to NestJS, however, is a _mental model_ for modules. That is, what role they play in application architecture and how they fit in with other non-Nest technologies that are a necessary part of the development process.

This article is an attempt to fill that missing _conceptual knowledge_ gap. I hope it helps both those new to NestJS, and those who are already productive Nest developers but perhaps still have a nagging doubt or two.

### Intro

We'll start by summarizing the concepts and techniques of _ES modules_ and _npm packages_ at a **basic level** to set a baseline and to ensure we are using _consistent terminology_. This is important because some of the concepts and terminology are overlapping and can easily get confused. We're on a mission to eliminate confusion! If those topics are old hat to you, by all means skip ahead to the NestJS-specific stuff!

We'll then go on to define the purpose of NestJS modules, review how they're constructed, and cover the relationship between modules, providers, and controllers. In short, we'll create that missing _mental model_ for how modules provide the cornerstone of NestJS application architecture.

After that, we'll discuss how all the conceptual pieces fit together nicely in a comprehensive mental model you can easily remember. We'll wrap up with a short discussion about _how to leverage these principals_ in your approach to application architecture.

### ES modules

This definitely **not** intended as a comprehensive review of the history and details of JavaScript modules! In fact, I plan to omit many details, but I promise that won't matter for our purposes. If you want to go learn the details, by all means do so -- there are plenty of great resources (I recommend [MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules)). The main purpose here is to get a few concepts straight, and then untangle some of the syntax you start running into right from your very first encounter with NestJS.

The basic components of ES modules you need to understand are:

- files, which are the basic _containers_ of code (note: strictly speaking these are called _modules_, but this is one potential source of confusion as these are **not** related to NestJS modules, and we can minimize confusion by avoiding use of that term in this context; _files_ works fine here)
- _bindings_, which can get exported from some files, and imported into others
- `import` and `export` keywords to assemble these pieces together

In terms of the canonical NestJS application, ES module syntax is seen in lines like:

```typescript
// in file src/app.service.ts
export class AppService...
```

and

```typescript
// in file src/app.module.ts
import { AppService } from './app.service';`
```

The important things to know here are:

- A _binding_ is the thing getting exported/imported. It can be a class, interface, constant, etc.
- Unless it is exported, any entity is _private_ to the file it lives in. This is good, because it means the class `MyFoo` in file `foo-a.ts` won't ever collide with the class `MyFoo` in file `foo-b.ts`. That is, unless you try to import one into the other. The point is that both you (the importer of a binding) and someone else (the exporter of a binding) have control over the visibility of code elements. In short, you can stay out of each others' way.

What are the takeaways for the NestJS developer?

1. For our purposes, we won't use the word _module_ to describe this feature; it's really not necessary, and only causes confusion if it's wrongly associated with the NestJS concept of a module.
2. We _will_ have to be mindful of the overloaded terms `import` and `export` which have separate and unrelated functions in ES module land and NestJS land. Don't worry, we'll make sure that's easy by the time this article is complete.
3. Whenever we want to access a _binding_ (class, interface, constant, etc.) from another file, we use ES `import`, like `import { AppService } from './app.service';`.
4. Whenever we want to share one, we use ES `export`, like `export class AppService { ... }`.
5. It all boils down to one easy-to-remember rule: if the entity isn't declared **in the file that wants to use it**, its binding has to be imported from some other file that exports it.
6. And finally, if you're using a modern development environment (like Visual Source Code) and TypeScript, errors at this level are usually quickly pointed out to you (and often fixed with the click of a mouse) in the editor itself. But at least now you have the foundation for understanding why such errors may occur.

### npm Packages

The Node Package Manager, npm, is a powerful system for storing and retrieving a group of related files, collectively known as a _package_, from a well-known global registry.

There's a bunch of infrastructure that makes this work (magically, all you really have to do is install `npm` to get it). All you usually need to know is that _npm_ can be used to **get more files onto your filesystem** that you can then use the ES module system to `import` from. And importing from these packages is _simple_ because they live in a standard location that Node.js knows how to find, so you don't even need to include filepath info, as you do with your own code imports (i.e., you get to omit the `./` part in `import { AppService } from './app.service'`).

In terms of the canonical NestJS application, an example of importing from a package installed via npm is:

```typescript
// in file src/app.module.ts
import { module } from `@nestjs/common`;
```

Having done that import, you now have full access to the `module` variable in `src/app.module.ts`, just like **any other import** we've already discussed.

What are the takeaways for the NestJS developer?

1. Almost always\*, you'll just use npm to get a bunch of code, in the form of a package, from the internet onto your local machine.
2. Once installed, Node.js makes these visible in a global way so that you can do the ES `imports` we covered earlier quite easily.
3. Generally, treat these packages as _black boxes_ that can only be accessed by importing bindings from them. Consider them "read only".

\* I say _almost always_ because, if you want to _publish_ a package, you need to understand a little more. I cover a lot of that in the article [Publishing NestJS Packages With npm](https://dev.to/nestjs/publishing-nestjs-packages-with-npm-21fm), but the bottom line is that, in addition to consuming npm packages, you can use them as a convenient way to collect up a bunch of related functionality and publish it for re-use, either inside or outside your organization.

### Putting These Pieces Together

The techniques we've seen so far are all about two things:

1. Where code physically lives in files.
2. How to get _bindings_ from otherwise private files exported and imported.

We've yet to deal with actually creating NestJS components (providers and controllers), and linking them together in meaningful ways (via modules). To provide an analogy for how these different aspects of NestJS development relate, let me tell you a little story.

When I was a kid, I loved doing jigsaw puzzles, and it ran in the family. My great aunt had a fascinating approach to doing them. She'd reach into the box, pull out a piece, and try to place it into the puzzle. If it didn't fit, she'd **put it back in the box**. Even as a kid, I found this both fascinating and bizarre. Nevertheless, she was **very** good (if a bit slow :wink:) at completing puzzles. My mom had a different technique. She'd "hoard" a bunch of similar pieces and try to complete her subsection of the puzzle off to the side.

Years later, when I introduced jigsaw puzzles to my own family, I figured it was time to set forth a few "rules" to preserve my sanity. Our family "policy" was to turn over all the pieces, find the edges, and work from there. Some of my kids liked to use my mom's approach of working on their subsection, but we insisted the do so on the same board so we could find the connections to the rest of the puzzle.

I like to think of the ES module and npm technology as a means of "getting all the pieces on the board, and getting related pieces close to each other". Doing that step has nothing to do with actually connecting puzzle pieces. It simply puts us in a position to do the constructive part -- assembling those pieces into a finished picture. In NestJS, that constructive part of the process is where module architecture -- building controllers and providers and connecting them up via modules -- comes into play.

The takeaway for a NestJS developer is that you _do_ need to work with ES modules and (usually also) npm to ensure that all of your pieces are on the board and can link up with each other, but that's **all** they do. They have **absolutely no other involvement** in how NestJS organizes your **application functionality or architecture**. Keep that in mind as we discuss **NestJS modules, imports and exports** and you'll be fine!

### NestJS Modules

OK, now we can get to the good stuff! Our goal here is to have a **mental model** of the architectural components of a NestJS application.

To summarize what we've learned so far, ES modules and npm packages are a way of **getting all the pieces on the board** so they can see each other, and can be assembled. They say **nothing** about how a NestJS application actually works.

Let's review the three major architectural components of a NestJS application:

1. **Injectables**. These are always classes, and their defining feature is that they participate in Dependency Injection. In other words, their lifecycle (instantiation, injection, destruction) is managed automatically by NestJS. This is different from other classes, whose lifecycle _you_ manage directly (e.g., with the `new` operator).
2. **Controllers**. These are also classes. Their lifecycle is also managed by NestJS. Their purpose, as you know, is to manage inbound requests from the outside world, and send outbound responses.
3. **Modules**. The key point about modules is this: they _overlay_ the other components. What do I mean by that? Let's break it down.

You may have noticed that _controllers_ and _injectables_ say **nothing** about a module in their declarations. There is no association _from_ an injectable or controller _to_ a module. Nothing that declares that "this provider belongs to this module". This might come as a surprise, but go ahead and take a look now -- you won't find any reference to a parent module anywhere in a controller or injectable declaration. The association works strictly in the other direction. Think of it like this: injectables and controllers, like any class, represent "potential objects" in the space of an application. They don't actually exist as objects **until they are declared as part of a module** (and, strictly speaking, until that module is instantiated during application bootstrapping). That's the central image to keep in mind in your mental model.

![Modules](./assets/modules1.png 'Modules')

So a module's purpose is to cause these potential objects to be instantiated, and to _anchor_ them to a particular context (scope) of your application. When Nest bootstraps your application, it uses the module's metadata to perform the necessary wiring (dependency injection, initialization, etc.) to make your application start working. Exactly _why_ the framework works this way is something we'll get to shortly.

First, let's take a little closer look at _injectables_ and _controllers_, in that order.

### NestJS Modules and Injectables

As discussed, injectable classes are purely _potential objects_ when they're declared. What makes them special is that you don't instantiate them, as with other classes, using the `new` operator. Instead -- and this is where the big picture starts to emerge -- their lifecycle is controlled by listing them in the `providers` property of a `@Module()` decorator\*. Injectables and providers are clearly linked together in a fundamental way. Let's quickly review what happens when we utilize that `providers` property.

\*Or, equivalently, in the `provider` property of a module metadata object returned via the dynamic module API.

As discussed in the [providers chapter](https://docs.nestjs.com/fundamentals/custom-providers#standard-providers), a _provider_ is a "recipe" for associating an injection token with a means of producing an instance of an injectable class. Significantly, that recipe appears in the `providers` property of a module metadata object. As a quick review, the following two fragments are equivalent:

```typescript
@Module({
  providers: [{
    provide: UsersService,
    useClass: UsersService
  }]
})
export class UsersModule { ... }
```

and, the shorthand version of the same:

```typescript
@Module({
  providers: [ UsersService ]
})
export class UsersModule { ... }
```

As we have learned, that provider recipe says "Hey Nest, whenever I try to reference the `UsersService` provider token in this module, give me an instance of the `UsersService` class. Now let's consider why _provider_ declarations appear in the `providers` property of a module. This is what binds modules and provider instances (we often call these instances "services") together. It's the `providers` property contents that give a module the information it needs to produce instances of injectable classes ("running services") when the module is bootstrapped.

> Note: We're assuming `SINGLETON` scope for the sake of this discussion. The impact of non-`SINGLETON` scope is discussed [here](https://docs.nestjs.com/fundamentals/injection-scopes).

![Modules](./assets/modules2.png 'Modules')

We can include the exact same provider object in another module, and we'll get another distinct instance of `UsersService`, scoped to that module.

![Modules](./assets/modules3.png 'Modules')

So here's the takeaway. Declaring a _provider_ in a module is what links an instance of the injectable to the scope of that module. Once again, in terms of your mental model, an injectable is just _potentially_ available when declared, but becomes anchored to and activated within a module when it appears as part of a provider recipe in a module's `providers` metadata (and that module is subsequently bootstrapped at runtime).

We often think of these activated injectables as _services_, and that's a helpful mental model. So now we can raise our level of abstraction for a moment, and think about modules as containers that run one or several services which have been activated for us by NestJS. This is a nice way to think about a running application. But understanding the mechanisms operating under the covers let's us fit the pieces together clearly. Let's do more with that.

For example, in any moderately interesting module, you'll likely have several services in play. Often there's a main service in a module (think `AuthService` in an `AuthModule`), but it needs other services to complete its work. We now have the vocabulary and mental model to understand how those services can interact. In order for `AuthService` to access something like `UsersService`, we know `UsersService` has to be visible inside `AuthModule`. We just covered one way to do that: include `UsersService` in the `providers` list of `AuthModule`'s metadata. That would certainly work, and would give us an instance of `UsersService` scoped to the `AuthModule`.

![Modules](./assets/modules4.png 'Modules')

However, what if `UsersService` is needed elsewhere in our app? Maybe we'd really rather have a single running instance of the service, and _share_ it across modules? That gives rise to the need to *export* providers from one Nest module, and *import* them into another. This is where the `imports` and `exports` properties in module metadata come into play. While a module (let's call it `ModuleA`) can see instances of classes listed in `ModuleA`'s `providers` list, it can **also** see providers from `ModuleB` if, and only if:

1. `ModuleA` lists `ModuleB` in it's module metadata `imports` property.
2. `ModuleB` lists some of its providers in its module metadata `exports` property.

In other words, `ModuleB` can provide stuff to itself, but also export some of its providers so `ModuleA` can see them (by simply importing `ModuleB`).

![Shared](./assets/shared-service.gif 'Shared Service')

Now, let's revisit our "pieces on the board" part of the analogy for a moment, and clear up one thing. Assuming you've taken care of the necessary scoping issues (provider visibility, imports, exports as discussed above), if you want your code to access an injected instance from some other class inside the module, you **must do an ES `import`** of the injectable class file where you are referencing it. Again, this step operates at the _puzzle-pieces-on-the-board layer_, not at the NestJS layer. In other words, when we want to access `CatsService` from another file, we need to do `import { CatsService } from './cats.service.ts';` before we can refer to it. This is a basic ES module requirement, **not a NestJS thing**.

What are the takeaways?

1. Repeating once again: injectables are classes declared without reference to a module, and aren't _activated_ until they are listed in a `providers` property of a module's metadata, at which time they are then associated with that module.
2. Modules comprise the scope (visibility) of their providers.
3. Modules can export providers, and import them from other modules that export them.
4. You still need to do ES imports and exports to make the code parts visible between files.

### Additional Module Details

Let's cover a few more advanced scenarios with modules.

#### Dynamic Modules

I tried to be careful above to refer to _module metadata_. The reason is that everything we discussed above applies equally to both _static_ modules and _dynamic modules_. An easy way to think about it is that _any_ module is defined by its _module metadata_. What is module metadata? Think of it as an object with these properties (taking some poetic license with types for clarity):

```typescript
{
  imports?: modules[],
  controllers?: controllers[],
  providers: providers[],
  exports?: providers[],
}
```

With static modules, these appear in the `@Module()` decorator property. With dynamic modules, they're returned from a static factory function. Otherwise, all of the concepts for how _injectables_, _providers_, and _modules_ interact apply to both types of modules.

There is, however, one interesting difference. If you think carefully about it, dynamic modules enable a sort of _nesting_ that is otherwise not possible. For example, take a look at this construct (taken from my [Asynchronous options providers article]()):

```typescript
@Module({
  imports: [MassiveModule.registerAsync({
    useExisting: ConfigService,
    imports: [ ConfigModule ]
  })]
})
```

Notice this code has a nested `imports` property. Let's consider that further for a moment.

#### Nested Module Contexts

What happens if, in the above code sample, the imported `ConfigModule` itself needed to be dynamically configured? We can automatically handle that case with:

```typescript
@Module({
  imports: [
    MassiveModule.registerAsync({
      useExisting: ConfigService,
      imports: [
        ConfigModule.register({
          useFolder: 'config'
        })
      ]
    })
  ]
})
```

We could imagine even further nesting:

```typescript
@Module({
  imports: [
    MassiveModule.registerAsync({
      useExisting: ConfigService,
      imports: [
        ConfigModule.registerAsync({
          useExisting: FileService,
          imports: [FileModule],
        })
      ]
    })
  ]
})
```

This Russian doll effect can, in principal, go as deep as needed. Let's examine what's happening here. Each layer is producing a dynamic module for the next layer up in the stack. The dynamic module being produced _has its own module context_. That means, if it needs to inject a provider that is _not_ in its module context, it has to import a module that supplies (provides and exports) that provider. Each such dynamic module is _isolated_ from its outer context. For example, in the construct above, the `MassiveModule` context doesn't have access to the `FileModule`'s providers (e.g., `FileService`). This module context isolation ensures that modules define a clean API that fits seamlessly with Nest's dependency injection system.

One reason for mentioning this (aside from how **astonishingly elegant** the NestJS architecture proves to be), is that you may sometimes encounter mysterious looking _"Nest can't resolve dependencies of the XYZService"_ messages. If you're using this nested imports technique, you may have to dig through the layers of context to identify where that missing dependency is happening.

#### Global Modules

So far, we've explicitly exported and imported providers between modules, with modules thereby providing an API for accessing providers. Kamil Myśliwiec, in a github issue, once gave this basic description of the purpose of modules: _"In Nest, modules compose providers and define their facades (export array)"_. This is a beautiful summary of what we've been talking about (and in so many fewer words! :smiley:).

Global modules break this model. Hence, they are both powerful and dangerous. Defining a module as global (with the `@Global()` decorator) makes any exported providers from that module globally available*. This is very handy, but it also can break encapsulation (the "module facade"), and make it hard to reason about provider imports and exports. Just be aware of this when utilizing global modules. As noted in the [documentation](https://docs.nestjs.com/modules#global-modules), *"The imports array is generally the preferred way to make the module's API available to consumers."*

\*A global module must still be imported once (usually in the [root module](https://docs.nestjs.com/modules), by convention usually `AppModule`) in order to become visible throughout the application.

### NestJS Modules and Controllers

Let's move on to controllers. They are by far easier to reason about. There's really only one main rule to keep in mind about controllers and modules:

**In order for Nest to mount a controller's routes, some module that is reachable from the root module must include the controller in the `controllers` property of its module metadata**.

By _reachable from the root module_ we mean that the controllers can be declared in one of the following:

- The root module.
- A module that the root module imports.
- A module that is transitively imported into the root module. For example, the root module imports `ModuleB`, which imports `ModuleC` which lists `Controller1` in its `controllers` module metadata.

It's worth mentioning that you don't (you can't) export controllers from a module. In essence, all Nest needs is a "path" (by way of exporting and importing modules) from the root module to a module containing controllers in order to initialize and mount them as route handlers.

The other thing to remember, of course, is that controllers often use providers to do things like interact with a database or an external API. And the normal rules about provider visibility apply -- a controller can see a provider that is in its module scope (which of course includes providers directly in the same module, as well as providers _imported_ into the module). These rules often result in the co-location of a controller with a closely related service, though that is of course not required.

### The Big Picture

We've covered a lot of ground here. We can now keep straight the various development technologies that have overlapping concepts and terminology. This should help us to organize the overall architecture of our apps, and to construct syntactically correct class definition files. To briefly summarize:

1. ES modules and npm "get the pieces on the board" -- they make it possible to reference code between files and in external libraries.
2. NestJS has its own module system. We can think of a running module as a container with active services running in it. We can think of the code defining a module as a namespace that provides scope for providers that are declared in it, and optionally imports and exports providers from other modules.
3. Controllers are simpler than providers, and simply need to be listed in the module metadata for some module that is reachable from the root module in order to be mounted.

#### Why Does This Matter?

Let's revisit our discussion about how modules define the scope of providers. This is important for two key reasons:

1. A module scope is like a namespace for a provider. If you use a provider called `Service1` only in `ModuleA`, you don't have to worry about `ModuleB` having something called `Service1`. This is helpful within your own code, but becomes critical in larger teams and/or when you're using library modules produced elsewhere.
2. A module becomes the _activation context_ for a provider. This is how we can have _dynamically configured modules_, like `@nestjs/jwt`, which allows you to _configure_ the `JwtService` when you import the module. See [here](https://docs.nestjs.com/fundamentals/async-providers) for more on asynchronous options providers and [here](https://docs.nestjs.com/fundamentals/dynamic-modules) for more on dynamic modules to understand the full power of these concepts. Modules are the "housing" for delivering providers that have these powerful capabilities.

#### How Should You Organize Modules, Providers and Controllers?

Answering this question becomes somewhat a matter of opinion and preference. There are no hard and fast rules. But a solid architecture should do its best to achieve several goals:

1. Loose coupling. Create providers, and APIs to configure those providers (i.e., dynamic modules), that don't know anything about their consumer. This makes them re-usable across modules and projects.
2. High cohesion. Where possible, keep the closely connected moving parts needed to deliver well-defined providers all within the same module. Do not break the loose coupling rule to create tight cohesion. This is the balancing act, where you need to decide how to break things up in the right way. Use modules and providers to achieve that.
3. Consistent API. Take a look at this [article about dynamic options providers]() and compare the registration API it presents to the [Custom providers API](). Having a consistent API across your providers and modules makes for coherence. NestJS has a beautiful, coherent structure and API, making it a joy to work with. If you learn to use modules effectively, and follow the examples you can find in modules like `@nestjs/jwt`, `@nestjs/passport`, etc., you'll be more likely to have an architecture that scales, is easily testable, and is maintainable. And a joy to work with. :smiley:

### Conclusion

In this article, I've tried to give you both a big picture "mental model" for how NestJS modules work, and how to leverage that understanding to create a [SOLID](https://en.wikipedia.org/wiki/SOLID) architecture for your application. Loosely speaking, modules are a way to "gather up" the main pieces (controllers and providers) of your application and glue them together in a coherent way. For "utility" type services (like logging, configuration, etc.), make use of [dynamic modules](https://docs.nestjs.com/fundamentals/dynamic-modules) to create a coherent API for re-using loosely coupled services within and across applications. I've offered a lot of opinion here, so I'd love to hear your comments and feedback! We're all trying to make NestJS-land a better and more productive place, so let's keep the dialog lively!
