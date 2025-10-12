---
title: Reducer organization - taking a step further
summary: Evolution of reducers in my Redux/NGRX apps that took place over the last two years
date: 2019-02-10
authors:
  - admin
tags:
  - tech revelation
  - web
---

## What are we gonna cover here?

We're going to overview evolution of reducers in my Redux/NGRX apps that took place over the last two years. Starting from vanilla `switch-case`, going to selecting a reducer from an object by key, finally settling with class-based reducers. We're not going to talk only about how, but also about why.

> If you're interested in working around too much boilerplate in Redux / NGRX you might want to check [this article](https://blog.goncharov.page/yet-another-guide-to-reduce-boilerplate-in-your-redux-ngrx-app) out.

> If you're already familiar with selecting a reducer from a map technique consider jumping right to [class-based reducers](#class-based-reducers).

## Vanilla switch-case

So let's take a look at an everyday task of creating an entity on the server asynchronously. This time I suggest we describe how we could create a new jedi.

```js
const actionTypeJediCreateInit = 'jedi-app/jedi-create-init'
const actionTypeJediCreateSuccess = 'jedi-app/jedi-create-success'
const actionTypeJediCreateError = 'jedi-app/jedi-create-error'

const reducerJediInitialState = {
  loading: false,
  // List of our jedi
  data: [],
  error: undefined,
}
const reducerJedi = (state = reducerJediInitialState, action) => {
  switch (action.type) {
    case actionTypeJediCreateInit:
      return {
        ...state,
        loading: true,
      }
    case actionTypeJediCreateSuccess:
      return {
        loading: false,
        data: [...state.data, action.payload],
        error: undefined,
      }
    case actionTypeJediCreateError:
      return {
        ...state,
        loading: false,
        error: action.payload,
      }
    default:
      return state
  }
}
```

Let me be honest, I've never used this kind of reducers in production. My reasoning is threefold:

- `switch-case` introduces some points of tension, leaky pipes, which we might forget to patch up in time at some point. We could always forget to put in `break` if do not do immediate `return`, we could always forget to add `default`, which we have to add to every reducer.
- `switch-case` has some boilerplate code itself which doesn't add any context.
- `switch-case` is O(n), [kind of](https://stackoverflow.com/questions/4442835/what-is-the-runtime-complexity-of-a-switch-statement). It is not a solid argument by itself because Redux is not very performant anyway, but it makes my inner perfectionist mad.

The logical next step which [Redux's official documentation](https://redux.js.org/recipes/reducing-boilerplate#generating-reducers) suggests to take is to pick a reducer from an object by key.

## Selecting a reducer from an object by key

The idea is simple. Each state transformation is a function from state and action and has a corresponding action type. Considering that each action type is a string we could create an object, where each key is an action type and each value is a function that transforms state (a reducer). Then we could pick a required reducer from that object by key, which is O(1), when we receive a new action.

```js
const actionTypeJediCreateInit = 'jedi-app/jedi-create-init'
const actionTypeJediCreateSuccess = 'jedi-app/jedi-create-success'
const actionTypeJediCreateError = 'jedi-app/jedi-create-error'

const reducerJediInitialState = {
  loading: false,
  data: [],
  error: undefined,
}
const reducerJediMap = {
  [actionTypeJediCreateInit]: (state) => ({
    ...state,
    loading: true,
  }),
  [actionTypeJediCreateSuccess]: (state, action) => ({
    loading: false,
    data: [...state.data, action.payload],
    error: undefined,
  }),
  [actionTypeJediCreateError]: (state, action) => ({
    ...state,
    loading: false,
    error: action.payload,
  }),
}

const reducerJedi = (state = reducerJediInitialState, action) => {
  // Pick a reducer by action type
  const reducer = reducerJediMap[action.type]
  if (!reducer) {
    // Return state unchanged if we did not find a suitable reducer
    return state
  }
  // Run suitable reducer if found one
  return reducer(state, action)
}
```

The cool thing here is that logic inside `reducerJedi` stays the same for any reducer, which means we can re-use it. There's even a small library, called [redux-create-reducer](https://github.com/kolodny/redux-create-reducer), which does exactly that. It makes the code looks like this:

```js
import { createReducer } from 'redux-create-reducer'

const actionTypeJediCreateInit = 'jedi-app/jedi-create-init'
const actionTypeJediCreateSuccess = 'jedi-app/jedi-create-success'
const actionTypeJediCreateError = 'jedi-app/jedi-create-error'

const reducerJediInitialState = {
  loading: false,
  data: [],
  error: undefined,
}
const reducerJedi = createReducer(reducerJediInitialState, {
  [actionTypeJediCreateInit]: (state) => ({
    ...state,
    loading: true,
  }),
  [actionTypeJediCreateSuccess]: (state, action) => ({
    loading: false,
    data: [...state.data, action.payload],
    error: undefined,
  }),
  [actionTypeJediCreateError]: (state, action) => ({
    ...state,
    loading: false,
    error: action.payload,
  }),
})
```

Nice and pretty, huh? Though this pretty still has a few caveats:

- In case of complex reducers we have to leave lots of comments describing what this reducer does and why.
- Huge reducer maps are hard to read.
- Each reducer has only one corresponding action type. What if I want to run the same reducer for several actions?

Class-based reducer became my shed of light in the kingdom of the night.

## Class-based reducers

This time let me start with whys of this approach:

- Class' methods will be our reducers and methods have names, which is a useful meta-information, and we could abandon comments in 90% of cases.
- Class' methods could be decorated which is an easy-to-read declarative way to match actions and reducers.
- We still could use a map of actions under the hood to have O(1) complexity.

If that's sounds like a reasonable list of reasons for you, let's dig in!

First of all, I would like to define what we want to get as a result.

```js
const actionTypeJediCreateInit = 'jedi-app/jedi-create-init'
const actionTypeJediCreateSuccess = 'jedi-app/jedi-create-success'
const actionTypeJediCreateError = 'jedi-app/jedi-create-error'

class ReducerJedi {
  // Take a look at "Class field delcaratrions" proposal, which is now at Stage 3.
  // https://github.com/tc39/proposal-class-fields
  initialState = {
    loading: false,
    data: [],
    error: undefined,
  }

  @Action(actionTypeJediCreateInit)
  startLoading(state) {
    return {
      ...state,
      loading: true,
    }
  }

  @Action(actionTypeJediCreateSuccess)
  addNewJedi(state, action) {
    return {
      loading: false,
      data: [...state.data, action.payload],
      error: undefined,
    }
  }

  @Action(actionTypeJediCreateError)
  error(state, action) {
    return {
      ...state,
      loading: false,
      error: action.payload,
    }
  }
}
```

Now as we see where we want to get we could do it step by step.

### Step 1. @Action decorator.

What we want to do here is to accept any number of action types and store them as meta-information for a class' method to use later. To do that we could utilize [reflect-metadata](https://github.com/rbuckton/reflect-metadata) polyfill, which brings meta-data functionality to [Reflect](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Reflect) object. After that this decorator would just attach its arguments (action types) to a method as meta-data.

```js
const METADATA_KEY_ACTION = 'reducer-class-action-metadata'

export const Action = (...actionTypes) => (target, propertyKey, descriptor) => {
  Reflect.defineMetadata(METADATA_KEY_ACTION, actionTypes, target, propertyKey)
}
```

### Step 2. Creating a reducer function out of a reducer class

As we know each reducer is a pure function that accepts a state and an action and returns a new state. Well, class is a function as well, but ES6 classes can not be invoked without `new` and we have to make an actual reducer out of a class with a few methods anyway. So we need to somehow transform it.

We need a function that would take our class, walk though each method, collect metadata with action types, build an reducer map and create a final reducer out of that reducer map.

Here's how we could examine each method of a class.

```js
const getReducerClassMethodsWthActionTypes = (instance) => {
  // Get method names from class' prototype
  const proto = Object.getPrototypeOf(instance)
  const methodNames = Object.getOwnPropertyNames(proto).filter(
    (name) => name !== 'constructor',
  )

  // We want to get back a collection with action types and corresponding reducers
  const res = []
  methodNames.forEach((methodName) => {
    const actionTypes = Reflect.getMetadata(
      METADATA_KEY_ACTION,
      instance,
      methodName,
    )
    // We want to bind each method to class' instance not to lose `this` context
    const method = instance[methodName].bind(instance)
    // We might have many action types associated with a reducer
    actionTypes.forEach((actionType) =>
      res.push({
        actionType,
        method,
      }),
    )
  })
  return res
}
```

Now we want to process the received collection into a reducer map.

```js
const getReducerMap = (methodsWithActionTypes) =>
  methodsWithActionTypes.reduce((reducerMap, { method, actionType }) => {
    reducerMap[actionType] = method
    return reducerMap
  }, {})
```

So the final function could look something like this.

```js
import { createReducer } from 'redux-create-reducer'

const createClassReducer = (ReducerClass) => {
  const reducerClass = new ReducerClass()
  const methodsWithActionTypes = getReducerClassMethodsWthActionTypes(
    reducerClass,
  )
  const reducerMap = getReducerMap(methodsWithActionTypes)
  const initialState = reducerClass.initialState
  const reducer = createReducer(initialState, reducerMap)
  return reducer
}
```

And we could apply it to our `ReducerJedi` class like this.

```js
const reducerJedi = createClassReducer(ReducerJedi)
```

### Step 3. Merging it all together.

```js
// We move that generic code to a dedicated module
import { Action, createClassReducer } from 'utils/reducer-class'

const actionTypeJediCreateInit = 'jedi-app/jedi-create-init'
const actionTypeJediCreateSuccess = 'jedi-app/jedi-create-success'
const actionTypeJediCreateError = 'jedi-app/jedi-create-error'

class ReducerJedi {
  // Take a look at "Class field delcaratrions" proposal, which is now at Stage 3.
  // https://github.com/tc39/proposal-class-fields
  initialState = {
    loading: false,
    data: [],
    error: undefined,
  }

  @Action(actionTypeJediCreateInit)
  startLoading(state) {
    return {
      ...state,
      loading: true,
    }
  }

  @Action(actionTypeJediCreateSuccess)
  addNewJedi(state, action) {
    return {
      loading: false,
      data: [...state.data, action.payload],
      error: undefined,
    }
  }

  @Action(actionTypeJediCreateError)
  error(state, action) {
    return {
      ...state,
      loading: false,
      error: action.payload,
    }
  }
}

export const reducerJedi = createClassReducer(ReducerJedi)
```

## Next steps

Here's what we missed:

- What if the same action corresponds to several methods? Current logic doesn't handle this.
- Could we add [immer](https://github.com/mweststrate/immer)?
- What if I use class-based actions? How could I pass an action creator, not an action type?

All of it with additional code samples and examples is covered with [reducer-class](https://github.com/fxlrnrpt/reducer-class).

I must say using classes for reducers is not an original thought. [@amcdnl](https://github.com/amcdnl) came up with awesome [ngrx-actions](https://github.com/amcdnl/ngrx-actions) quite a while ago, but it seems like he's now focused on [NGXS](https://github.com/ngxs/store), not to mention I wanted more strict typing and decoupling from Angular-specific logic. [Here's a list of key differences between reducer-class and ngrx-actions.](https://github.com/fxlrnrpt/reducer-class#how-does-it-compare-to-ngrx-actions)

> If you like the idea of using classes for your reducers, you might like to do the same for your action creators. Take a look at [flux-action-class](https://github.com/fxlrnrpt/flux-action-class).

Hopefully, you found something useful for your project. Feel free to communicate your feedback to me! I most certainly appreciate any criticism and questions.