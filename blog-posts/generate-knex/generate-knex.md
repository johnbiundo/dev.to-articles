---
published: true
title: "Build a NestJS Module for Knex.js (or other resource-based libraries) in 5 Minutes"
description: "tags: nestjs, nest, knexjs, sql, node.js"
series:
cover_image: "https://user-images.githubusercontent.com/6937031/65389455-5ca1d800-dd0b-11e9-9a8a-aefb41895009.png"
canonical_url:
---

_John is a member of the NestJS core team_

Ever wanted to integrate your favorite library into NestJS? For example, while Nest has broad built-in support for database integration, what if you want to use your own favorite library and no Nest package exists? Well, why not build your own?

This might seem like a daunting task at first. But if you've been [following my blog posts](https://dev.to/johnbiundo), you saw a _design pattern_ for NestJS dynamic modules in my [last post](https://dev.to/nestjs/advanced-nestjs-how-to-build-completely-dynamic-nestjs-modules-1370) that points you in the right direction. OK, you say, but it _still_ seems like a lot of work to integrate a library.

I have good news! Using the awesome power of the Nest CLI, you can **generate a complete, custom dynamic module template, following NestJS best practices, with a single command!** You can then literally have your library integrated in about 5 minutes! This technique works for many _resource-based_ libraries that export what I'll call an _API object_, which naturally work well with Nest's built-in _singleton provider_ model for sharing a global resource. Follow along with such an adventure as we build a quick integration to [Knex.js](http://knexjs.org/) below.

And stay tuned for more on the wonders of the vastly under-appreciated NestJS CLI. Built on top of the Angular CLI, the potential for this tool to revolutionize the way you use Nest is limitless. I have lots more planned for this topic in the near future!

### Intro

In my [How to build completely dynamic NestJS modules](https://dev.to/nestjs/advanced-nestjs-how-to-build-completely-dynamic-nestjs-modules-1370) article last week, I covered a design pattern that is used throughout standard Nest packages like `@nestjs/passport`, `@nestjs/jwt`, and `@nestjs/typeorm`. The point of the article was two-fold:

- Provide a roadmap to a basic pattern used within common NestJS packages to help you be a better consumer of those packages.
- Encourage Devs to think about how to make use of this pattern in their own code to enable more easily composable modules.

Some might consider the pattern a bit complex, and I wouldn't necessarily argue. However, gaining a good understanding of it is useful, and helps in the quest for mastery of some core concepts of Nest, especially around leveraging the _module system_ and _dependency injection_. Nevertheless, there's a fair amount of boilerplate code in the pattern. Wouldn't it be great to simplify the process?

Buckle up! NestJS CLI custom schematics to the rescue! :rocket:

### What We'll Build

In this tutorial, we'll build a module that exports a direct API to the full [Knex.js](http://knexjs.org/) library. This is a powerful DB integration library used widely across the Node.js ecosystem. Here's what we'll do. The following represent the _exact same steps_ you can use to integrate any other basic callable API (for example, [ioredis](https://github.com/luin/ioredis), [Cassandra](https://github.com/datastax/nodejs-driver#readme), [Neo4J](https://github.com/neo4j/neo4j-javascript-driver#readme), [Elasticsearch](https://www.elastic.co/blog/new-elasticsearch-javascript-client-released), [LevelDb](https://github.com/Level/levelup) to name just a few).

1. Use the [dynpkg](https://github.com/nestjsplus/dyn-schematics#use-case-1-generating-a-standalone-package) _custom schematic_ to generate a customized package (the schematic automates the [dynamic module pattern](https://dev.to/nestjs/advanced-nestjs-how-to-build-completely-dynamic-nestjs-modules-1370) I've been discussing above).
2. Fill in a few quick details to customize the package for Knex.js.
3. Use the auto-generated test client to demonstrate that it works.

Thanks to the power of the Nest CLI, coupled with a _custom schematic_, we can complete these steps in about 5 minutes! As a bonus, we'll cover a few more advanced features at the end of this article, including how to publish the generated package on npm with a single click.

If you want to see a completed version of the steps presented in the remainder of this article (a _fully useable NestJS/Knex.js module_), you can find a [github repo with the completed code here](https://github.com/nestjsplus/knex).

### Nest CLI and Schematics

You probably already use the Nest CLI on a regular basis. `nest new myProject` and `nest generate controller user` are common examples. They use the `@nestjs/schematics` package that ships with the CLI out-of-the-box. What's very cool is that you can easily use other schematics to do highly customized work. A feature common to all schematics is that they _understand_ your project, and have a smart way of scaffolding new architectural components and wiring them into your project. Think of an individual schematic as a blueprint, and of the CLI machinery as a _set of standards_ for how each blueprint works. Because of this, new schematics "just work", and inherit all of the features provided by the CLI and its standard capabilities.

You can write your own schematics, and I plan to show you how in some upcoming blog posts. For now, I've built one that you can use to do the library integration project we've undertaken today. Let's get started!

To use any external schematics, you of course need to install them. One important note: you **must** install a schematics collection as a global package, due to the way the CLI works.

#### Installing a Custom Schematics Collection

Schematics are packaged as _collections_ and can be bundled up as npm packages, so installing them is simple. You can do so now with:

```bash
npm install @nestjsplus/dyn-schematics -g
```

#### Using a Custom Schematics Collection

Using schematics from a custom collection is straightforward. The standard Nest CLI command structure looks like this:

> nest _commandOrAlias_ [-c schematicCollection] _requiredArg_ [options]

- _`commandOrAlias`_ is: `new` or `generate` (alias: `g`) or `add`
- `schematicCollection` is optional, and defaults to the built-in NestJS schematics; you can optionally specify a globally installed npm package
- _`requiredArg`_ is the architectural element being generated/added, e.g., `controller`, `module`, or soon, in our case `dynpkg`
- `options` are global to all schematics; they can be `--dry-run`, `--no-spec`, or `--flat`

Once it's installed, you use the [@nestjsplus/dyn-schematics](https://github.com/nestjsplus/dyn-schematics) package like this:

1. Make sure you're in the folder you want to have as the parent of the project. With this schematic, we're creating a _new_ complete nest package, meaning it's a new standalone project. So it will create a folder using the `name` you provide, and put all of the component parts inside that folder.
2. Run the schematic with the CLI

```bash
nest g -c @nestjsplus/dyn-schematics dynpkg nest-knex
```

This runs using the custom schematics collection from `@nestjsplus/dyn-schematics` and specifying the `dynpkg` schematic to execute (the schematic identifies the _thing_ to generate - in this case, a _dynamic module package_ identified as `dynpkg`), and giving it a `name` of `nest-knex`. If you want to see what this would do **without adding files** to your filesystem, just add `--dry-run` at the end of the command.

This should prompt you with the question `Generate a testing client?`. Answer yes to this to make testing easier. You'll see the following output, and you'll then be able to open this project in your IDE from the `nest-knex` folder.

```bash
► nest g -c @nestjsplus/dyn-schematics dynpkg nest-knex
? Generate a testing client? Yes
CREATE /nest-knex/.gitignore (375 bytes)
CREATE /nest-knex/.prettierrc (51 bytes)
CREATE /nest-knex/README.md (2073 bytes)
CREATE /nest-knex/nest-cli.json (84 bytes)
CREATE /nest-knex/package.json (1596 bytes)
CREATE /nest-knex/tsconfig.build.json (97 bytes)
CREATE /nest-knex/tsconfig.json (430 bytes)
CREATE /nest-knex/tslint.json (426 bytes)
CREATE /nest-knex/src/constants.ts (54 bytes)
CREATE /nest-knex/src/index.ts (103 bytes)
CREATE /nest-knex/src/main.ts (519 bytes)
CREATE /nest-knex/src/nest-knex.module.ts (1966 bytes)
CREATE /nest-knex/src/nest-knex.providers.ts (262 bytes)
CREATE /nest-knex/src/nest-knex.service.ts (1111 bytes)
CREATE /nest-knex/src/interfaces/index.ts (162 bytes)
CREATE /nest-knex/src/interfaces/nest-knex-module-async-options.interface.ts (532 bytes)
CREATE /nest-knex/src/interfaces/nest-knex-options-factory.interface.ts (194 bytes)
CREATE /nest-knex/src/interfaces/nest-knex-options.interface.ts (409 bytes)
CREATE /nest-knex/src/nest-knex-client/nest-knex-client.controller.ts (732 bytes)
CREATE /nest-knex/src/nest-knex-client/nest-knex-client.module.ts (728 bytes)
```

At this point, you can install the generated code:

```bash
cd nest-knex
npm install
```

Now start up the app with:

```bash
npm run start:dev
```

If you now browse to [http://localhost:3000](http://localhost:3000) you should get:

> Hello from NestKnexModule!

Great! At this point, we've scaffolded the _framework_ of our dynamic module, and we're ready to begin the Knex.js integration.

### Integrating with Knex.js

You now have a project scaffolded to do the library integration. We have to make just a few edits to perform the actual integration. Let's talk them through:

1. We need to install `knex` as a dependency. Note, in order to **run** this project, you'll need access to a live SQL database. Knex.js doesn't bundle any database libraries, so you'll need to install the appropriate one ([you can read more here](http://knexjs.org/#Installation-node)). In this tutorial, I'll use PostgreSql, which I have available on localhost. If you do not have a local DB available, you can consider [using a docker setup](https://docs.docker.com/samples/library/postgres/).
2. We need a way to provide the configuration options to the Knex.js API.
3. We need to call the Knex.js API in the way it wants to be initialized, returning a handle to a `knex` object which we can use to access Knex.js features.
4. To test things out, we can use the client controller that was auto-generated by the schematic.
5. Once it works, we can use this package directly. Or we can publish the package to a registry (e.g., an internal package registry or publicly to npmjs.com)

#### Install Dependencies

The `knex` package is required. You will also need a database API library to **run** the module. Below, I use `pg` for PostgreSql, but you can choose whatever you want (from the [list here](http://knexjs.org/#Installation-node)).

```bash
cd nest-knex
npm install knex pg
```

#### Knex.js Options

As discussed in the [dynamic modules article](https://dev.to/nestjs/advanced-nestjs-how-to-build-completely-dynamic-nestjs-modules-1370#async-options-providers-usecase), we provide options to the service using an _async options provider_. The code to do all this has been scaffolded for you, but to pass the appropriate options in a native TypeScript way, we need to modify the `NestKnexOptions` interface. This is contained in the file `src/interfaces/nest-knex-options.interface.ts`.

We want to describe the available Knex.js options that will be passed to the library API. The easiest way to do this is to use the types already [provided by Knex.js](https://github.com/tgriesser/knex/blob/master/types/index.d.ts#L1590). Since this interface is exported by the `knex` package, we can simply import it, and our job is nearly done! We'll simply alias it to make it visible to our generated package. Open up `src/interfaces/nest-knex-options.interface.ts` and edit it to look like this:

```typescript
// src/interfaces/nest-knex-options.interface.ts
import { Config } from 'knex';

export interface NestKnexOptions extends Config {}
```

#### Knex Connections

The job of our module is pretty simple: connect to the API and return a re-usable `knex` object for interacting with the database. In the case of PostgreSql, `knex` (on top of `pg`) returns a _connection pool_, which means we can connect once and return a _singleton_ `knex` object that will automatically balance multiple queries across the connection pool. This works _seamlessly_ with NestJS and its notion of singleton providers!

> Note: I'm less familiar with Knex than other DB libraries, and also less familiar with other DBs like MySQL, so if you are using this module for such databases, be sure you understand the right pattern for sharing a `knex` connection object within your application. This topic is **not** a NestJS issue -- it's all about the architecture of the Knex.js library and the database client library.

Now we'll implement a simple pattern to return our `knex` object. Open up `src/nest-knex.service.ts`. Have a quick look to see what the generated code looks like, then replace that code with the following:

```typescript
// src/nest-knex.service.ts
import { Injectable, Inject, Logger } from '@nestjs/common';
import { NEST_KNEX_OPTIONS } from './constants';
import { NestKnexOptions } from './interfaces';

const Knex = require('knex');

interface INestKnexService {
  getKnex();
}

@Injectable()
export class NestKnexService implements INestKnexService {
  private readonly logger: Logger;
  private _knexConnection: any;
  constructor(@Inject(NEST_KNEX_OPTIONS) private _NestKnexOptions: NestKnexOptions) {
    this.logger = new Logger('NestKnexService');
    this.logger.log(`Options: ${JSON.stringify(this._NestKnexOptions)}`);
  }

  getKnex() {
    if (!this._knexConnection) {
      this._knexConnection = new Knex(this._NestKnexOptions);
    }
    return this._knexConnection;
  }
}
```

Almost all of this is unchanged from the generated boilerplate. We simply added a property to store our connection object (`_knexConnection`), and a `getKnex()` method to instantiate the object the first time it's requested, and then cache it for future use.

#### Test the Module

As mentioned, you'll need a local database instance available to test the module. If you don't have one handy, consider [using docker](https://docs.docker.com/samples/library/postgres/) to do so. It's easy!

The `@nestjsplus/dyn-schematics` `dynpkg` schematic built a small test _client module_ for us (assuming you answered `yes` to the prompt). We can use this to quickly test our `nest-knex` module.

First, open `src/nest-knex-client/nest-knex-client.module.ts` and add the needed Knex.js options in the `register()` method. Adjust yours appropriately:

```typescript
// src/nest-knex-client/nest-knex-client.module.ts
...
@Module({
  controllers: [NestKnexClientController],
  imports: [
    NestKnexModule.register({
      client: 'pg',
      connection: {
        host: 'localhost',
        user: 'john',
        password: 'mypassword',
        database: 'nest',
        port: 5432,
      },
    }),
  ],
})
...
```

Now, open `src/nest-knex-client/nest-knex-client.controller.ts` and plug in some queries. Of course this is not really a great design pattern (invoking database services directly from your controller), but is really just a quick test that the Knex.js integration works. In reality, you'll want to delegate any such access to true NestJS _services_ as per NestJS best practices.

Here's an idea of what you can try. This test relies on the following database table being available (syntax below works with PostgreSql, but may require slight tweaks for other databases):

```sql
CREATE TABLE cats
(
   id     serial    NOT NULL,
   name   text,
   age    integer,
   breed  text
);

ALTER TABLE cats
   ADD CONSTRAINT cats_pkey
   PRIMARY KEY (id);
```

And here's a sample of what you could try in your test controller. Note that you have the full power of Knex.js available via the `knex` object. See here for [lots more interesting samples](http://knexjs.org/#Builder) of things you can do with Knex. Since you have a handle to the `knex` object, all of those _Knex Query Builder_ methods should just work! In this admittedly silly example, we are both creating and querying some cats in our index route, using the `knex` object to access the database.

```typescript
// src/nest-knex-client/nest-knex-client.controller.ts
import { Controller, Get } from '@nestjs/common';
import { NestKnexService } from '../nest-knex.service';

@Controller()
export class NestKnexClientController {
  constructor(private readonly nestKnexService: NestKnexService) {}

  @Get()
  async index() {
    const knex = this.nestKnexService.getKnex();
    const newcat = await knex('cats').insert({
      name: 'Fred',
      age: 5,
      breed: 'tom cat',
    });

    const cats = await knex.select('*').from('cats');

    return cats;
  }
}
```

Now, fire up the app with `npm run start:dev`, and browse to `http://localhost:3000`, and you should see the results of your query!

If you want to test the module with a separate app -- really demonstrating the power of the reusable library module you just built -- [read on](#publish-the-package) to see how to upload it as an npm package in one step. And for convenience, you can download a [full client app that imports the module here](https://github.com/nestjsplus/knex-cats) and use it to test your newly minted Knex.js module. In fact, [knex-cats](https://github.com/nestjsplus/knex-cats) is a fully working example, so if you've been reading along without coding, that's the easiest way to see the finished product.

### Bonus Section

Hopefully I kept my promise of showing you how quickly you can generate a dynamic NestJS module that wraps an external resource library in just a few minutes! We're done -- you have a full-fledged Knex.js API at your disposal now! The queries we did in the previous section are just the tip of the iceberg, as Knex.js has [a boat load of cool features](http://knexjs.org/#Builder).

Let's cover a couple more topics to round out the discussion.

#### A Better API

As mentioned, it's not good practice to access the `knex` object directly from our controller. We can make things a little bit better by creating a service to house our database logic. We can also make our Knex.js module a little easier to use by adding a higher level API. Let's do that. We're going to add a _provider_ that let's us directly inject the `knex` object into any service. You'll see how this cleans the API up in a moment.

First, create a file called `nest-knex-connection.provider.ts` inside the `src` folder. Add the following code:

```typescript
// src/nest-knex-connection.provider.ts
import { KNEX_CONNECTION } from './constants';
import { NestKnexService } from './nest-knex.service';

export const connectionFactory = {
  provide: KNEX_CONNECTION,
  useFactory: async nestKnexService => {
    return nestKnexService.getKnex();
  },
  inject: [NestKnexService],
};
```

To follow best practices for providers, we are using the `KNEX_CONNECTION` constant as our provider injection token. Be sure to add this line to `src/constants.ts`:

```typescript
export const KNEX_CONNECTION = 'KNEX_CONNECTION';
```

See what we're doing here? We're binding an injection token (`KNEX_CONNECTION`) to a factory so that we can use our `knex` API object directly, avoiding the need to instantiate the service. For example, once we complete one more step (below), we'll be able to access the `knex` object in our controller as follows (notice the slightly simplified technique between _old_ and _new_ in the sample code):

```typescript
// src/nest-knex-client/nest-knex-client.controller.ts
import { Controller, Inject, Get } from '@nestjs/common';
import { KNEX_CONNECTION } from '../constants';

@Controller()
export class NestKnexClientController {
  // new
  constructor(@Inject(KNEX_CONNECTION) private readonly knex) {}
  // old
  // constructor(private readonly nestKnexService: NestKnexService) {}

  @Get()
  async index() {
    // following line no longer needed
    // const knex = this.nestKnexService.getKnex();
    const newcat = await this.knex('cats').insert({
      name: 'Fred',
      age: 5,
      breed: 'tom cat',
    });

    const cats = await this.knex.select('*').from('cats');

    return cats;
  }
}
```

To wire this new provider into our module, open up `src/nest-knex.module.ts` and make the following changes:

1. Import the `connectionFactory`
2. Add the `connectionFactory` to the `@Module` metadata `providers` and `exports` properties. This section of the file will now look like this:

```typescript
import { connectionFactory } from './nest-knex-connection.provider';

@Global()
@Module({
  providers: [NestKnexService, connectionFactory],
  exports: [NestKnexService, connectionFactory],
})
```

The complete implementation of our best practices for database access would also have us create a **service** that accesses the `KNEX_CONNECTION` provider, rather than doing that in a controller, but I'll leave that as an exercise for the reader. :wink:

#### Dynamic Registration (with a Config Factory)

In the [dynamic modules article](https://dev.to/nestjs/advanced-nestjs-how-to-build-completely-dynamic-nestjs-modules-1370#async-options-providers-usecase), we talked about using _asynchronous options providers_. By that, we meant that rather than use a _statically declared_ object containing our connection params like this:

```typescript
    NestKnexModule.register({
      client: 'pg',
      connection: {
        host: 'localhost',
        user: 'john',
        password: 'mypassword',
        database: 'nest',
        port: 5432,
      },
    }),
```

We'd instead like to supply our connection options in a dynamic fashion. Good news! This is **already built-in** to the generated package. Let's test it out. For simplicity, we'll just use an _in-place_ factory, but you can use the full power of _class-based_, _factory based_, and _existing_ providers. To test it, we'll update the registration of the `NestKnexModule` in `src/nest-knex-client/nest-knex-client.module.ts` to look like this:

```typescript
// src/nest-knex-client/nest-knex-client/module.ts
    NestKnexModule.registerAsync({
      useFactory: () => {
        return {
          debug: false,
          client: 'pg',
          connection: {
            host: 'localhost',
            user: 'john',
            password: 'mypassword',
            database: 'nest',
            port: 5432,
          },
        };
      },
    }),
```

It's not a terribly exciting example, but you get the idea. You can take advantage of any of the asynchronous provider methods to supply the configuration data to the module. If you check out the [knex-cats](https://github.com/nestjsplus/knex-cats) sample, you'll see some more robust use of a config module to configure our Knex module dynamically.

#### Publish the Package

If you read my earlier [Publishing NestJS Packages with npm](https://dev.to/nestjs/publishing-nestjs-packages-with-npm-21fm) article, you may have noticed that the _structure_ of this package (e.g., the `package.json` file, the `tsconfig.json` file, the presence of the `index.ts` file in the root folder) follows that pattern exactly. As a result, publishing this package is as simple as this (assuming you have an `npmjs.com` account, that is):

```bash
npm publish
```

Seriously, that's it!

Of course, you can always use this ready-made [@nestjsplus/knex]() package, built _exactly_ as describe in this article.

### Not Just External APIs!

The [@nestjsplus/dyn-schematics package](https://github.com/nestjsplus/dyn-schematics) not only supports generating a standalone package, as described so far, but also generating a regular dynamic module _inside an existing project_. I won't cover that in detail here, but the use-case is simple. Let's say you're building a _ConfigModule_ for use within a project. Simply scaffold it with the `dynmod` schematic (instead of `dynpkg`), and it will generate a new dynamic module _into your existing project_, fully ready for you to implement the details of your `register()` and `registerAsync()` static methods.

This works _exactly_ like the normal CLI commands for adding elements like _controllers_, _services_, and _modules_. In fact, its behavior is very parallel to `nest generate module`. It builds the _dynamic module_ and all its parts (which look quite similar to what we've seen in this article, minus the top level Nest application components) in a folder inside your project, and _wires them up to the rest of the project_ just as the normal `module` schematic does. Read more about it at the github page for [@nestjsplus/dyn-schematics](https://github.com/nestjsplus/dyn-schematics). This use-case (adding a dynamic module to an existing project) is [covered here](https://github.com/nestjsplus/dyn-schematics#use-case-2-adding-a-dynamic-module-to-an-existing-project). You can easily give it a quick _dry run_ inside an existing Nest project with:

```bash
nest g -c @nestjsplus/dyn-schematics dynmod myNewModule --dry-run
```

### Conclusion

If you've been following this series, you've previously learned a powerful architectural pattern for building modular NestJS services and modules, and composing them into applications. Now, with the power of schematics, you can _automatically generate_ the boilerplate code to implement this powerful pattern. You can use this schematic to customize your own internal modules, or to easily integrate external APIs. In future articles, I'll show you how to build your own schematics to extend the power of NestJS and the NestJS CLI even further!

Feel free to ask questions, make comments or suggestions, or just say hello in the comments below. And join us at [Discord](https://discord.gg/G7Qnnhy) for more happy discussions about NestJS. I post there as _Y Prospect_.

```

```
