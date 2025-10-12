---
title: node-config made type-safe
summary: node-config has been serving the Node.js community as pretty much the default config solution for many years. Its simplistic, yet powerful design helped it to spread like a virus across multiple JS libraries. Yet those very design choices don't always play nice with new strictly-typed kids on the block. Like TypeScript. How could we keep using our favorite config tool and stay on the type-safe side of things?
date: 2019-04-12
authors:
  - admin
tags:
  - tech revelation
  - web
---

[node-config](https://github.com/lorenwest/node-config) has been serving the Node.js community as pretty much the default config solution for many years. Its simplistic, yet powerful design helped it to spread like a virus across multiple JS libraries. Yet those very design choices don't always play nice with new strictly-typed kids on the block. Like TypeScript. How could we keep using our favorite config tool and stay on the type-safe side of things?

## Step 1: Make an interface for your config

Supposedly, you have a `config` folder somewhere in your project, which in the most simple case has this structure:

- `default.ts`
- `production.ts`

> Yes, [node-config](https://github.com/lorenwest/node-config) speaks TypeScript out-of-the-box!

Let's consider a case of writing a config for an app, which creates new worlds populated only by cats.

Our default config could look like this:

```ts
// default.ts
const config = {
  // Default config assumes a regular 4-pawed 1-tailed cat
  cat: {
    pawsNum: 4,
    tailsNum: 1,
  },
}

module.exports = config
```

Our production config could be this:

```ts
// production.ts
const configProduction = {
  // In production we create mutant ninja cats with 8 paws
  cat: {
    pawsNum: 8,
  },
}

module.exports = configProduction
```

Basically, our production config is always a subset of our default one. So we could create an interface for our default config and use a partial ([DeepPartial](https://stackoverflow.com/questions/45372227/how-to-implement-typescript-deep-partial-mapped-type-not-breaking-array-properti/49936686#49936686) to be true) of that interface for our production config.

Let's add `constraint.ts` file with the interface:

```ts
// constraint.ts
export interface IConfigApp {
  cat: {
    pawsNum: number
    tailsNum: number
  }
}
// We'll need this type for our production config.
// Alternatively, you can use ts-essentials https://github.com/krzkaczor/ts-essentials
export type DeepPartial<T> = {
  [P in keyof T]?: T[P] extends Array<infer U>
    ? Array<DeepPartial<U>>
    : T[P] extends ReadonlyArray<infer U>
    ? ReadonlyArray<DeepPartial<U>>
    : DeepPartial<T[P]>
}
```

Then we could use it in our `default.ts`:

```ts
// default.ts
import { IConfigApp } from './constraint'

const config: IConfigApp = {
  // Default config assumes a regular 4-pawed 1-tailed cat
  cat: {
    pawsNum: 4,
    tailsNum: 1,
  },
}

module.exports = config
```

And in our `production.ts`:

```ts
// production.ts
import { IConfigApp, DeepPartial } from './constraint'

const configProduction: DeepPartial<IConfigApp> = {
  // In production we create mutant ninja cats with 8 paws
  cat: {
    pawsNum: 8,
  },
}

module.exports = configProduction
```

## Step 2: Make `config.get` type-safe

So far we resolved any inconsistencies between a variety of our configs. But `config.get` still returns `any`.

To fix that let's add another typed version of `config.get` to its prototype.

Assuming your projects has the folder `config` in its root and the code of your app in the folder `src`, let's create a new file at `src/config.service.ts`

```ts
// src/config.service.ts
import config from 'config'

// The relative path here resolves to `config/constraint.ts`
import { IConfigApp } from '../config/constraint'

// Augment type definition for node-config.
// It helps TypeScript to learn about uor new method we're going to add to our prototype.
declare module 'config' {
  interface IConfig {
    // This method accepts only first-level keys of our IConfigApp interface (e.g. 'cat').
    // TypeScript compiler is going to fail for anything else.
    getTyped: <T extends keyof IConfigApp>(key: T) => IConfigApp[T]
  }
}

const prototype: config.IConfig = Object.getPrototypeOf(config)
// Yep. It's still the same `config.get`. The real trick here was with augmenting the type definition for `config`.
prototype.getTyped = config.get

export { config }
```

Now we can use `config.getTyped` anywhere in our app, importing it from `src/config.service`.

> Be aware, we no longer can do `caonfig.getTyped('cat.pawsNum')`. It's impossible to make it type-safe.

> To make `import config from 'config'` work set `"esModuleInterop": true` under `compilerOptions` in your `tsconfig.json`.

It could look like this in our `src/app.ts`:

```ts
// src/app.ts
import { config } from './config.service'

const catConfig = config.getTyped('cat')
```

[**Live demo**](https://repl.it/@aigoncharo/node-config-made-type-safe-demo)

> You might want to use [tsconfig-paths](https://github.com/dividab/tsconfig-paths) to avoid deep relative imports.

Hopefully, you've found something useful for your project. Feel free to communicate your feedback to me! I most certainly appreciate any criticism and questions.