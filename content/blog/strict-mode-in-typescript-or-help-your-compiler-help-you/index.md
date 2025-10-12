---
title: Strict mode in TypeScript || help your compiler help you
summary: Recently, at Hazelcast we have migrated our Management Center to TypeScript. Not just TypeScript, but the strictest TypeScript there is. If you are interested in why we decided to enable all the strict flags and benefits they provide, welcome to this post.
date: 2021-07-28
authors:
  - admin
tags:
  - tech revelation
  - web
---

Recently, at Hazelcast we have migrated our Management Center to TypeScript. Not just TypeScript, but the strictest TypeScript there is. If you are interested in why we decided to enable all the strict flags and benefits they provide, welcome to this post.

## Prelude

50k lines of JavaScript code of a React app. A small team of 2, later 3. 166 bugs filed from 2018-04-24 to 2019-04-23. On average, we produced 9.66 bugs per month.
As developers, we are naturally lazy. Well, maybe, not truly lazy, rather highly unmotivated to do trivial repetitive work. Like fixing a color of a button or another simple, yet annoying bug. So what is the ultimate solution?
Option 1 - never-dying classics. One of us works on state-of-the-art AI for the next couple of decades. Another one puts everything they have into physics and creates a time travel machine over, roughly, the same amount of time. The last one goes above and beyond to prove that our efforts are worth it to the management for the next 30 years or so. In the end, we pack that coolest AI on a flash drive, jump into the time travel machine, go back to the present time, and make the AI do the dirty work of bug fixing for us. While this plan is almost spotless, we have faced one major problem - nobody wanted to be the last guy that deals with the management. So we had to find a different way.
Option 2 - create fewer bugs. "What" here is simple. The problem is with "how". So how?
The holy grail of robust enterprise-grade frontend - TypeScript! 

## Act 1

Enabling TypeScript on a project was a breeze. You just put `any` here and there, the rest of the `any`s are inferred. As result, from 2019-04-24 to 2020-01-29 we have observed 90 bugs filed. On average it is 10 bugs per month. Wait. WHAT?!
It was a moment of revelation that TypeScript, being an awesome language, does not magically slay dragons for you. Especially, if you misuse it. 
We have scratched our heads for a bit, thoroughly examined documentation, and concluded that what we need is "strict" TypeScript. After enabling strict flags one by one, our average SoC (Speed of Crapping all over the code base) went down to... 3.3 bugs per month! 3 times less! 

**Intermediate conclusion #1**. "Loose" TypeScript did not make any difference for us, but "strict" TypeScript made our code significantly more robust.


## Act 2

So what flags exactly did we enable? 

> TLDR; We have enabled all of them ;)

TypeScript compiler has over 90 various flags. 7 of them are strict (or more, depending on what you call a strict flag). 1 flag to rule them all. Let's go over 7 original strict flags and see if they add any value.

Flags in question:
- noImplicitAny
- noImplicitThis
- alwaysStrict
- strictBindCallApply
- strictNullChecks
- strictPropertyInitialization
- strictFunctionTypes

### noImplicitAny

#### Flag disabled

```ts
const fn = (val) => val + 1
console.log(fn('100'))
```

**Result**: compiles without a question.

#### Flag enabled

```ts
const fn = (val) => val + 1
// PARAMETER ‘VAL’ IMPLICITLY HAS AN ‘ANY’ TYPE
console.log(fn('100'))
```

**Result**: fails on `val`.

#### Value

10/10

### noImplicitThis

Say, we have a `Cat`. `Cat` can eat and poop.

```ts
class Cat {
   consumedFood = 0

   eat(food: number) {
       this.consumedFood += food
   }

   poop() {
       return this.consumedFood
   }
}
```

Even better. Say, we have an `EnterpriseCat`. It can eat and poop via `poopFactory`.

```ts
class EnterpriseCat {
   consumedFood = 0

   eat(food: number) {
       this.consumedFood += food
   }

   poopFactory() {
       return function () {
           return this.consumedFood
       }
   }
}
```

#### Flag disabled

```ts
class EnterpriseCat {
   consumedFood = 0

   eat(food: number) {
       this.consumedFood += food
   }

   poopFactory() {
       return function () {
           return this.consumedFood
       }
   }
}
console.log(new EnterpriseCat().poopFactory()())
```

**Result**: compiles without a question. In runtime it fails because of the lost context.

#### Flag enabled

```ts
class EnterpriseCat {
   consumedFood = 0

   eat(food: number) {
       this.consumedFood += food
   }

   poopFactory() {
       return function () {
           // THIS IMPLICITLY HAS TYPE ‘ANY’ BECAUSE IT DOESN’T HAVE A TYPE ANNOTATION
           return this.consumedFood
       }
   }
}
console.log(new EnterpriseCat().poopFactory()())
```

**Result**: fails on `this`.

#### Value

7/10

### alwaysStrict

Treats every file as if it had "use strict" appended on the top.

#### Value

5/10

### strictBindCallApply

#### Flag disabled

```ts
function foo(a: number, b: string): string {
   return a + b;
}

console.log(foo.apply(undefined, [10]))
```

**Result**: compiles without a question.

#### Flag enabled

```ts
function foo(a: number, b: string): string {
   return a + b;
}

// ARGUMENT OF TYPE ‘[NUMBER]’ IS NOT ASSIGNABLE TO PARAMETER OF TYPE ‘[NUMBER, STRING]’
console.log(foo.apply(undefined, [10]))
```

**Result**: fails on `apply`.

#### Value

5/10

### strictNullChecks

#### Flag disabled

```ts
const doEpicShi_Stuff = (val: string) => val
console.log(doEpicShi_Stuff(null))
```

**Result**: compiles without a question.

#### Flag enabled

```ts
const doEpicShi_Stuff = (val: string) => val
// ARGUMENT OF TYPE ‘NULL’ IS NOT ASSIGNABLE TO PARAMETER OF TYPE ‘STRING’
console.log(doEpicShi_Stuff(null))
```

**Result**: fails on `doEpicShi_Stuff`.

#### Value

9/10

### strictPropertyInitialization

#### Flag disabled

```ts
class Cat {
   name: string
}
console.log(new Cat().name)
```

**Result**: compiles without a question. Prints `undefined` which is kind of unexpected based on the type.

#### Flag enabled

```ts
class Cat {
   // PROPERTY ‘NAME’ DOES NOT HAVE AN INITIALIZER AND IS NOT DEFINITELY ASSIGNED IN THE CONSTRUCTOR
   name: string
}
console.log(new Cat().name)
```

**Result**: fails on `name`.

#### Value

8/10

### strictFunctionTypes

Probably, the most misunderstood and underappreciated flag.

#### Flag disabled

```tsx
// Here it is string | null
const catName = (state): string | null => state.name

// Here it is string
interface CatProps {
  name: string
}
const CatInternal = ({ name }: CatProps) => <div>{name}</div>

// Nevertheless, it works
export const Cat = connect((state) => ({
  name: catName(state)
}))(CatInternal)

```

**Result**: compiles without a question. Even without `strictNullChecks`.

Why?

#### A little of the type-related theory

Say, you have a class `Animal`. `Dog` is its direct child. Class `Greyhound` is a child of the `Dog`. So we have the following sequence:

```
Greyhound < Dog < Animal
```

Say, you have a function that accepts a callback and returns a boolean. We expect the callback to accept a `Dog` and return a `Dog`.

```
f: (Dog -> Dog) -> boolean
```

You know, let's make things a bit more fun.

Say, you have a `Primate`. `Primate` experienced thousands of years of evolution to become a `Human`. `Human` drank a gazillion of smoothies and even attended one or two meetups to become an `FE Developer`. So we have the following sequence:

```
FE Developer < Human < Primate
```

Now imagine, that the function above (`f`) tries to answer a popular question: "How to become a software engineer good enough for Google?" So we need to consider different callbacks that somehow transform one sort of `Human` into a different sort of `Human` and return if that way of the transformation is good enough for Google. In other words:

```
isGoodForGoogle: (transform: Human -> Human) -> boolean
```

**Option 1**

Read lots of geek-oriented Reddit. It will definitely help any `Human` to quickly transition into a `Primate`. So the sequence is `Human -> Primate`.

Let's see it in action:

```
`Human` -> (`Human` -> `Primate`) -> `Primate` -> `Primate.read()`
```

Since `transform` is expected to return a `Human`, it is perfectly reasonable to expect that `Human` to read. Can a `Primate` read? No. So this option does nto works for us.

**Option 2**

Pass a 4-hour JS online "from-zero-to-hero" crash course. Probably, it will leave us right where we started. So `Primate -> Primate` it is.

Let's see it in action:

```
`Human` -> (`Primate` -> `Primate`) -> `Primate` -> `Primate.read()`
```

Same thing with option 2.

**Option 3**

Work 16 hours a day as a frontend engineer. Since we are already doing frontend with this option, it looks like `FE Developer -> FE Developer`.

Let's see it in action:

```
`Human` -> (`FE Developer` -> `FE Developer`) -> `FE Developer`
```

At first glance, it looks alright, but wait. We can pass any human, right? Can we pass a `BE Developer`? 

```
`BE Developer` -> (`FE Developer` -> `FE Developer`) -> `FE Developer`
```

Now what if our transform `FE Developer` -> `FE Developer` under the hood does something like `FE Developer.configureWebpack()`? Can a `BE Developer` deal with webpack? I would say it is beyond any reasonable expectations.

So this option does not work for us as well.

**Option 4**

Constantly invest in your own education over months, years, and decades. I guess it should give us `Human -> FE Developer`.

Let's see it in action:

```
`Human` -> (`Human` -> `FE Developer`) -> `FE Developer`
```

Yay! This is it. Moreover, it could even be `Primate -> FE Developer`, not `Human -> FE Developer`. Using proper terminology, this option checks the argument *contravariantly* and checks the return type *covariantly*. Guess what? Without `strictFunctionTypes` TypeScript checks the arguments *bivariantly* that creates a bug we saw earlier.

**Real word**

```ts
{ name: string } < { name: string | null }
```

or

```ts
A < A | null
```

**Intermediate conclusion #2**. TypeScript checks the arguments *bivariantly* by default. `strictFunctionTypes` makes it check the arguments *contravariantly*.

#### Value

8/10

## Postlude 

As we can see, the greatest value is provided by the flags that require the most refactoring. So after enabling the most challenging and the most rewarding flags it makes next to no effort to enable them all. Especially, since the awesome TypeScript team took care of us and added another flag called **strict**. It enables all of the flags above. In Management Center we went down precisely that path. We enabled the most complex flags first and enabled all of them in the end as it was extremely small overhead.

> We left flags like `noImplicitOverride`, `noImplicitReturns`, `noUncheckedIndexedAccess`, `noPropertyAccessFromIndexSignature` and other out of scope of this article because they are not managed by the `strict` uber-flag.