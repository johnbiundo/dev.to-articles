---
published: false
title: "Understanding NestJS Modules"
description: "tags: nestjs, nest, dependency injection, providers"
series:
canonical_url:
---

*John is a member of the NestJS core team*

Modules are considered basic building blocks of NestJS applications.  Before you know it, you're plunged right into using them.  The [NestJS docs](https://docs.nestjs.com) cover *how* to use modules comprehensively, but what I find is sometimes missing for people new to NestJS is a *mental model* for modules, and how they fit in with other parts of the development process. This article is an attempt to fill that missing *conceptual knowledge*.  I hope it helps both those new to NestJS, and those who have are already productive Nest developers, but might have a nagging doubt or two.

### Intro

In this article, we'll first cover some basics that are, strictly speaking, outside the scope of NestJS.  Feel free to jump past those intro sections to review some of the Nest-specific content.

We'll start by covering the topics of *ES modules* and *npm packages* at a **basic level** to set a baseline and ensure we are using consistent terminology.  We need to do this because some of the concepts and terminology are overlapping and can easily get confused. We're on a mission to eliminate confusion! Again, if those topics are old had to you, by all means skip ahead to the NestJS-specific stuff!

We'll then go on to define the purpose of NestJS modules, review how they're constructed, and cover the relationship between modules, providers, and controllers.

We'll wrap up by making sure all the conceptual pieces fit together nicely in a *mental model* you can easily remember.

### ES Modules

This is most definitely **not** a comprehensive review of the history and details of JavaScript modules! In fact, I plan to omit many details, and I may even get a few nuances wrong (but I promise that won't matter for our purposes).

I hope I don't provoke the wrath of the JavaScript gods by taking this loose approach, and if you want to go learn the details, by all means do so -- there are plenty of great resources.  The main purpose here is to get a few concepts straight, and then untangle some of the syntax you start running into from your very first day with NestJS.

So, the basic components of ES Modules you need to understand are:
- files, which are the basic *containers* of code
- *bindings*, which can get exported from some files, and imported into others
- `import` and `export` keywords to assemble these pieces together

In terms of the canonical NestJS application, these are lines like

```typescript
// in file src/app.service.ts
export class AppService...
```
and

```typescript
// in file src/app.module.ts
import { AppService } from './app.service';`
```

The only things you really need to know here are:
- A *binding* is the thing getting exported/imported.  It can be a class, interface, constant, etc..
- Unless it is exported, any entity is *private* to the file it lives in.  This is good, because it means the class `myFoo` in `foo-a.ts` won't ever collide with the class `myFoo` in `foo-b.ts`.  That is, unless you try to import one into the other.  The point is that both you (the exporter of a binding) and someone else (the importer of a binding) have control over the visibility of code elements.  In short, you can stay out of each others' way.
- These mechanics are considered a part of the ES Module system.  Unfortunately, while they use the terms *module* and *import*, these are **completely separate** from the same-named concepts in NestJS.  In fact, to avoid confusion, we won't even use the term "module" when referring to stuff going on at this level as we go forward.  Simply think in terms of files with code and using imports and exports to *link together* otherwise private bits of code. That's pretty much it.

What are the takeaways for the NestJS developer?
1. For our purposes, we won't use the word *module* to describe this feature; it's really not necessary, and only causes confusion.
2. We *will* have to be mindful of the overloaded terms `import` and `export`.  Don't worry, we'll make sure that's easy by the time this article is complete.
3. Whenever we want to access a *binding* (class, interface, constant, etc.) from another file, we use ES `import`, like `import { AppService } from './app.service';`.
4. Whenever we want to share one, we use ES `export`, like `export class AppService { ... }`.
5. It can all be boiled down to one rule: if the binding doesn't exist **in the file that wants to use it**, it has to be imported from some other file that exports it.

### npm Packages

The Node Package Manager, npm, is an awesome system for storing and retrieving a group of files, known as a *package*, from a well-known global registry.

There's a bunch of infrastructure that makes this work (magically, all you really have to do is install `npm` to get it). At the end of the day, all you usually need to know is that *npm* can be used to **get more files onto your filesystem** that you can then use the ES Module system to `import` from.  And importing from these packages is *simple* because they live in a standard location that Node.js knows how to find, so you don't even need to include filepath info, as you do with your own code imports (i.e., the `./` part in `import { AppService } from './app.service'`).

In terms of the canonical NestJS application, an example of importing from a package installed via npm is:

```typescript
// in file src/app.module.ts
import { module } from `@nestjs/common`
```
Having done that import, like **any other import** we've already discussed, you now have full access to the `module` binding inside `src/app.module.ts`.

What are the takeaways for the NestJS developer?
1. Almost always*, you'll just use *npm* to get a bunch of code, in the form of a **package**, from the internet onto your local machine.
2. Once installed, Node.js makes these visible in a global way so that you can do the ES `imports` we covered earlier quite easily.

\* I say almost always because, if you want to *publish* a package, you need to understand a little more. I cover a lot of that in [](), but the bottom line is that, in addition to consuming npm packages, you can use as a convenient way to collect up a bunch of related functionality and publish it for re-use, either inside or outside your organization.

### Putting these Pieces Together

What we've seen so far is all about two things:
1. Where code physically lives in files
2. How to get *bindings* from otherwise private files exported and imported

To make sense of all this, let me tell you a little story.  Keep in mind that we're oversimplifying a bit to get the main point across.  We'll get a little more sophisticated and technically accurate later in this article. But here goes.

Once upon a time, Cindy started writing a JavaScript program to organize her daily tasks. At first, the program was simple, and she just used it to print out her daily task list.  Soon, she wanted to share her program with her friend Fred, who also asked her to make a tiny change.  Before long, Cindy's program was being used by many friends, and had grown an impressive list of features.  When Cindy wanted to make a change, she opened up her now rather large program file in Notepad, and made the changes. Cindy soon saw that other people had built "components" that made her programming easier, and so she copied and pasted those components into her enormous file.  While it was convenient to have all this code in one file, Cindy soon found she was running into problems - it was too hard to find the code she needed, she was repeating the same bits of code in many places, and annoyingly, other people were using the same names as her for their JavaScript variables. Cindy was forced to learn stuff like *npm* and *ES Modules* to deal with this.  What she found was that these tools **broke her ginormous file** into pieces, but still let her **treat them as if they were all virtually in her single gargantuan file**.

The moral of the story is that the purpose of ES Modules and npm packages is to let you take a bunch of pieces of code, and treat them as if they were all in one place.  Kind of like taking all the pieces of a jigsaw puzzle and putting them on the table so they can easily be fit together.

That's it! And the big takeaway, for a NestJS developer, is that you *do* need to work with these capabilities to ensure that all of your pieces are on the board and can link up with each other, but that's **all** they do.  They have **no other involvement** in how NestJS organizes your **application functionality or architecture**. Keep that in mind as we discuss **NestJS modules, imports and exports** and you'll be fine!

### NestJS Modules

OK, now we can get to the good stuff!  Our goal here is to have a **mental model** of the architectural components of a NestJS application.

To summarize what we've learned so far (taking a **bit** of poetic license), ES Modules and npm packages are a way of **getting all the pieces on the board** so they can see each other, and can be assembled.  They say **nothing** about how a NestJS application actually works.

Let's review the three major architectural components of a NestJS application:

1. **Injectables**.  These are always classes, and their defining feature is that they participate in Dependency Injection.  In other words, their lifecycle (instantiation, destruction) can be managed automatically by NestJS.  We'll separately discuss the concept of a **Provider** in a moment.  It's closely related to, but not identical, to an injectable.
2. **Controllers**.  These are also classes.  Their lifecycle is also managed by NestJS.  Their purpose, as you know, is to manage inbound requests from the outside world, and send outbound responses.
3. **Modules**.  The key point about modules is this: they *overlay* the other components.  What do I mean by that?  Let's break it down.

You may have noticed that *controllers* and *injectables* say **nothing** about a module in their declarations.  There is **no association** from an injectable or controller **to** a module.  Go ahead and take a look now - you won't find any reference to a module anywhere in a controller or injectable definition file.  The association works strictly in the other direction.  Think of it like this: injectables and controllers "float freely" in the space of an application.  They aren't useful **until they are declared as part of a module**.  That's the central image to keep in mind in your mental model.

So a module's purpose is to *anchor* a controller or injectable to a particular context of your application.  Once that's done, Nest knows how to bootstrap your application and perform the necessary wiring (dependency injection, initialization, etc.) to make your application start working.

Let's look at these in turn.  We'll start with injectables.

### NestJS Modules and Injectables

As discussed, injectable classes kind of "float in space" when they're declared. It's only when you list one in the `providers` property of an `@Module()` decorator (or equivalently, return one via the Dynamic Module API) that it becomes functional in an application.  Let's quickly review what happens with that `providers` property.

As discussed in [xxx](), a *provider* is a "recipe" for associating an injection token with an instance of an injectable. The following two fragments are equivalent:

```typescript
@Module({
  providers: [{
    provide: CatsService,
    useClass: CatsService
  }]
})
export class AppModule { ... }
```

and, the shorthand version of the same:

```typescript
@Module({
  providers: [ CatsService ]
})
export class AppModule { ... }
```

Once declared in this fashion, a single instance of `CatsService` becomes available for use within the module*.  We can say that `CatsService` is available within the *module scope* of the `AppModule`. At runtime, it becomes an *injected instance* within the module scope.

\*We're assuming `SINGLETON` scope here for the sake of this discussion.  The impact of non-`SINGLETON` scope is discussed [here]().

We can do the exact same thing in another module, and we'll get another distinct instance of `CatsService`, scoped to that module.  So, declaring a provider in a module is **the thing that activates** an injectable within the scope of that module.

Where things get interesting is that we may have other injectable classes and controllers that are active within that module's scope. Any such injectable class or controller can access any other injected instance (one that has been activated in that module's scope by being declared as a provider).

A couple of things to point out here:
1. If you want to access the injected instance from another class inside the module, you **must do an ES `import`** of the injectable class file where you are referencing it.  This step operates at the *puzzle-pieces-on-the-board layer*, not at the NestJS layer.  In other words, when we want to access `CatsService` from another file, we need to do `import { CatsService } from './cats.service.ts';` before we can refer to it.  This is a basic ES Module requirement, **not a NestJS thing**.
2. Injectables that have not been *provided* (appeared in the `providers` metadata) in the current module are **not** visible to other controllers and injectable classes in that module.

Let's add one more rule to the list here.  While a module (let's call it `ModuleA` can see instances of classes listed in `ModuleA`'s `providers` list, it can **also** see providers from `ModuleB` if, and only if:
1. `ModuleA` lists `ModuleB` in it's module metadata `imports` property.
2. `ModuleB` lists some of its providers in its module metadata `exports` property.

In other words, `ModuleB` can provide stuff to itself, but also export some of its providers so `ModuleA` can see them (by simply importing `ModuleB`).

What are the takeaways?
1. Repeating once again: injectables are classes declared without reference to a module, and aren't *activated* until they are listed in a `providers` property of a module's metadata, at which time they are then associated with that module.
2. Modules define the scope (visibility) for providers.
3. Modules can export providers, and import them from other modules that export them.
4. You still need to do ES imports and exports to make the code parts visible between files.

### Additional Module Details

#### Why does this matter?

We talk about modules defining the scope of providers.  This is important for two key reasons:

1. A module scope is like a namespace for a provider.  If you use a provider called `Service1` only in `ModuleA`, you don't have to worry about `ModuleB` having something called `Service1`.  This is helpful within your own code, but becomes critical in larger teams and/or when you're using library modules produced elsewhere.
2. A module becomes the *activation context* for a provider. This is how we can have *dynamically configured modules*, like `@nestjs/jwt`, which allows you to *configure* the `JwtService` when you import the module. See [x]()  and [y]() for more on dynamic modules and asynchronous options providers to understand the full power of these concepts.  Modules are the "housing" for delivering providers that have these powerful capabilities.

#### GLobal Modules



### NestJS Modules and Controllers



### The Big Picture








