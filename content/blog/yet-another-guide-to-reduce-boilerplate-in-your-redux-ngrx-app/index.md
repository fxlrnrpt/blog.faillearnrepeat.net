---
title: Yet another guide to reduce boilerplate in your Redux (NGRX) app
summary: Several ways/tips/tricks/ancient black magic rituals to reduce boilerplate in our overwhelmed-with-boilerplate Redux (and NGRX!) apps
date: 2019-01-04
authors:
  - admin
tags:
  - tech revelation
  - web
---

## What are we gonna cover here?

Several ways/tips/tricks/ancient black magic rituals to reduce boilerplate in our overwhelmed-with-boilerplate Redux (and NGRX!) apps I came up with over the years of first-hand production experience.

Let me be honest with you, guys. I wanted to speak just about my new micro-library [flux-action-class](https://github.com/fxlrnrpt/flux-action-class) at first, but it seems like everybody's been complaining how tech blogs more and more look like Twitter, how everyone wants some meaningful long reading and etc. So I thought: "What the heck? I got some experience and best practices of my own which I spilled some sweat and blood over. Maybe, it could help some people out there. Maybe, people out there could help me to improve some of it."

## Identifying boilerplate

Let's take a look at a typical example of how to make AJAX requests in Redux. In this particular case let's imagine we wanna get a list of cats from the server.

```js
import { createSelector } from 'reselect'

const actionTypeCatsGetInit = 'CATS_GET_INIT'
const actionTypeCatsGetSuccess = 'CATS_GET_SUCCESS'
const actionTypeCatsGetError = 'CATS_GET_ERROR'

const actionCatsGetInit = () => ({ type: actionTypeCatsGetInit })
const actionCatsGetSuccess = (payload) => ({ type: actionTypeCatsGetSuccess: payload })
const actionCatsGetError = (error) => ({ type: actionTypeCatsGetError, payload: error })

const reducerCatsInitialState = {
  error: undefined,
  data: undefined,
  loading: false,
}
const reducerCats = (state = reducerCatsInitialState, action) => {
  switch (action.type) {
    case actionTypeCatsGetInit:
      return {
        ...state,
        loading: true,
      }
    case actionCatsGetSuccess:
      return {
        error: undefined,
        data: action.payload,
        loading: false,
      }
    case actionCatsGetError:
      return {
        ...data,
        error: action.payload,
        loading: false,
      }
    default:
      return state
  }
}

const makeSelectorCatsData = () =>
  createSelector(
    (state) => state.cats.data,
    (cats) => cats,
  )
const makeSelectorCatsLoading = () =>
  createSelector(
    (state) => state.cats.loading,
    (loading) => loading,
  )
const makeSelectorCatsError = () =>
  createSelector(
    (state) => state.cats.error,
    (error) => error,
  )
```

_If you're wondering why I have selector factories (makeSelector...) take a look [here](https://redux.js.org/recipes/computing-derived-data#computing-derived-data)_

_I'm leaving out side effect handling on purpose. It's a topic for a whole different article full of teenager's anger and criticism for the existing ecosystem :D_

This code has several weak spots:

- Action creators are unique objects themselves but we still need action types for serialization purposes. Could we do better?
- As we add entities we keep duplicating the same logic for flipping `loading` flag. Actual server data and the way we want to handle it may change, but logic for `loading` is always the same. Could we get rid of it?
- Switch statement is O(n), [kind of](https://stackoverflow.com/questions/4442835/what-is-the-runtime-complexity-of-a-switch-statement), (which is not a solid argument by itself because Redux is not very performant anyway), requires couple extra lines of code for each `case` and switches can not be easily combined. Could we figure out something more performant and readable?
- Do we really need to keep an error for each entity separately?
- Using selectors is a good idea. This way we have an abstraction over our store and can change its shape without breaking the whole app by just adjusting our selectors. Yet we have to create a factory for each selector due to how memoizaion works. Is there any other way?

## Tip 1: Get rid of action types

Well, not really. But we can make JS generate them for us!

Let's take a minute here to think why we even need action types? Obviously, to help the reducer somehow differentiate incoming actions and change our state accordingly. But does it really have to be a string? If only we had a way to create objects (actions) of certain types... Classes to the rescue! We most definitely could use classes as action creators and do `switch` by type. Like this:

```js
class CatsGetInit {}
class CatsGetSuccess {
  constructor(responseData) {
    this.payload = responseData
  }
}
class CatsGetError {
  constructor(error) {
    this.payload = error
    this.error = true
  }
}

const reducerCatsInitialState = {
  error: undefined,
  data: undefined,
  loading: false,
}
const reducerCats = (state = reducerCatsInitialState, action) => {
  switch (action.constructor) {
    case CatsGetInit:
      return {
        ...state,
        loading: true,
      }
    case CatsGetSuccess:
      return {
        error: undefined,
        data: action.payload,
        loading: false,
      }
    case CatsGetError:
      return {
        ...data,
        error: action.payload,
        loading: false,
      }
    default:
      return state
  }
}
```

All good, but here's a thing... We can no longer serialize and deserialize our actions. They are no longer simple objects with prototype of Object. All of them have unique prototypes which actually makes switching over `action.constructor` work. Dang, I liked the idea of serializing my actions to a string and attaching it to bug reports. So could we do even better?

Actually, yes! Luckily each class has a name, which is a string, and we could utilize them. So for the purposes of serialization each action needs to be a simple object with field `type` (please, take a look [here](https://github.com/redux-utilities/flux-standard-action) to learn what else any self-respecting action should have). We could add field `type` to each one of our classes which would use class' name.

```js
class CatsGetInit {
  constructor() {
    this.type = this.constructor.name
  }
}
const reducerCats = (state, action) => {
  switch (action.type) {
    case CatsGetInit.name:
      return {
        ...state,
        loading: true,
      }
    //...
  }
}
```

It would work, but this way we can not prefix our action types as [this](https://github.com/erikras/ducks-modular-redux) great proposal suggests (actually, I like its [successor](https://github.com/alexnm/re-ducks) even more). To work around prefixing we should stop using class' name directly. What we could do is to create a static getter for type and utilize it.

```js
class CatsGetInit {
  get static type () {
    return `prefix/${this.name}`
  }
  constructor () {
    this.type = this.constructor.type
  }
}
const reducerCats = (state, action) => {
  switch (action.type) {
    case CatsGetInit.type:
      return {
        ...state,
        loading: true,
      }
    //...
  }
}
```

Let's polish it a little to avoid code duplication and add one more assumption to reduce boilerplate even further: if action is an error action `payload` must be an instance of `Error`.

```js
class ActionStandard {
  get static type () {
    return `prefix/${this.name}`
  }
  constructor(payload) {
    this.type = this.constructor.type
    this.payload = payload
    this.error = payload instanceof Error
  }
}

class CatsGetInit extends ActionStandard {}
class CatsGetSuccess extends ActionStandard {}
class CatsGetError extends ActionStandard {}

const reducerCatsInitialState = {
  error: undefined,
  data: undefined,
  loading: false,
}
const reducerCats = (state = reducerCatsInitialState, action) => {
  switch (action.type) {
    case CatsGetInit.type:
      return {
        ...state,
        loading: true,
      }
    case CatsGetSuccess.type:
      return {
        error: undefined,
        data: action.payload,
        loading: false,
      }
    case CatsGetError.type:
      return {
        ...data,
        error: action.payload,
        loading: false,
      }
    default:
      return state
  }
}
```

At this point it works perfectly with NGRX, but Redux is complaining about dispatching non-plain objects (it validates the prototype chain). Fortunatelly, JS allows us to return an arbitrary value from the constructor and we do not really need our actions to have a prototype.

```js
class ActionStandard {
  get static type () {
    return `prefix/${this.name}`
  }
  constructor(payload) {
    return {
      type: this.constructor.type,
      payload,
      error: payload instanceof Error
    }
  }
}

class CatsGetInit extends ActionStandard {}
class CatsGetSuccess extends ActionStandard {}
class CatsGetError extends ActionStandard {}

const reducerCatsInitialState = {
  error: undefined,
  data: undefined,
  loading: false,
}
const reducerCats = (state = reducerCatsInitialState, action) => {
  switch (action.type) {
    case CatsGetInit.type:
      return {
        ...state,
        loading: true,
      }
    case CatsGetSuccess.type:
      return {
        error: undefined,
        data: action.payload,
        loading: false,
      }
    case CatsGetError.type:
      return {
        ...data,
        error: action.payload,
        loading: false,
      }
    default:
      return state
  }
}
```

Not to make you guys copy-paste `ActionStandard` class and worry about its reliability I created a [small library called flux-action-class](https://github.com/fxlrnrpt/flux-action-class), which already got all that code covered with tests with 100% code coverage, written in TypeScript for TypeScript and JavaScript projects.

## Tip 2: Combine your reducers

The idea is simple: use [combineReducers](https://redux.js.org/api/combinereducers) not only for top level reducers, but for combining reducers for `loading` and other stuff. Let the code speak for itself:

```js
const reducerLoading = (actionInit, actionSuccess, actionError) => (
  state = false,
  action,
) => {
  switch (action.type) {
    case actionInit.type:
      return true
    case actionSuccess.type:
      return false
    case actionError.type:
      return false
  }
}

class CatsGetInit extends ActionStandard {}
class CatsGetSuccess extends ActionStandard {}
class CatsGetError extends ActionStandard {}

const reducerCatsData = (state = undefined, action) => {
  switch (action.type) {
    case CatsGetSuccess.type:
      return action.payload
    default:
      return state
  }
}
const reducerCatsError = (state = undefined, action) => {
  switch (action.type) {
    case CatsGetError.type:
      return action.payload
    default:
      return state
  }
}

const reducerCats = combineReducers({
  data: reducerCatsData,
  loading: reducerLoading(CatsGetInit, CatsGetSuccess, CatsGetError),
  error: reducerCatsError,
})
```

## Tip 3: Switch away from switch

Use objects and pick from them by key instead! Pick a property of an object by key is O(1) and it looks much cleaner if you ask me. Like this:

```js
const createReducer = (initialState, reducerMap) => (
  state = initialState,
  action,
) => {
  // Pick a reducer from the object by key
  const reducer = reducerMap[action.type]
  if (!reducer) {
    return state
  }
  // Run the reducer if present
  return reducer(state, action)
}

const reducerLoading = (actionInit, actionSuccess, actionError) =>
  createReducer(false, {
    [actionInit.type]: () => true,
    [actionSuccess.type]: () => false,
    [actionError.type]: () => false,
  })

class CatsGetInit extends ActionStandard {}
class CatsGetSuccess extends ActionStandard {}
class CatsGetError extends ActionStandard {}

const reducerCatsData = createReducer(undefined, {
  [CatsGetSuccess.type]: () => action.payload,
})
const reducerCatsError = createReducer(undefined, {
  [CatsGetError.type]: () => action.payload,
})

const reducerCats = combineReducers({
  data: reducerCatsData,
  loading: reducerLoading(CatsGetInit, CatsGetSuccess, CatsGetError),
  error: reducerCatsError,
})
```

I suggest we refactor `reducerLoading` a little bit. With introduction of reducer maps it makes sense to return a reducer map from `reducerLoading` so we could easily extend it if needed (unlike switches).

```js
const createReducer = (initialState, reducerMap) => (
  state = initialState,
  action,
) => {
  // Pick a reducer from the object by key
  const reducer = state[action.type]
  if (!reducer) {
    return state
  }
  // Run the reducer if present
  return reducer(state, action)
}

const reducerLoadingMap = (actionInit, actionSuccess, actionError) => ({
  [actionInit.type]: () => true,
  [actionSuccess.type]: () => false,
  [actionError.type]: () => false,
})

class CatsGetInit extends ActionStandard {}
class CatsGetSuccess extends ActionStandard {}
class CatsGetError extends ActionStandard {}

const reducerCatsLoading = createReducer(
  false,
  reducerLoadingMap(CatsGetInit, CatsGetSuccess, CatsGetError),
)
/*  Now we can easily extend it like this:
    const reducerCatsLoading = createReducer(
      false,
      {
        ...reducerLoadingMap(CatsGetInit, CatsGetSuccess, CatsGetError),
        ... some custom stuff
      }
    )
*/
const reducerCatsData = createReducer(undefined, {
  [CatsGetSuccess.type]: () => action.payload,
})
const reducerCatsError = createReducer(undefined, {
  [CatsGetError.type]: () => action.payload,
})

const reducerCats = combineReducers({
  data: reducerCatsData,
  loading: reducerCatsLoading),
  error: reducerCatsError,
})
```

[Redux's official documentation mentions this](https://redux.js.org/recipes/reducing-boilerplate#generating-reducers), but for some reason I saw lots of people still using switch-cases. There's already a [library](https://github.com/kolodny/redux-create-reducer) for `createReducer`. Do not hesitate to use it.

## Tip 4: Have a global error handler

It's absolutely not necessary to keep an error for each entity individually, because in most cases we just need to display an error dialog or something. The same error dialog for all of them!

Create a global error handler. In the most simple case it could look like this:

```js
class GlobalErrorInit extends ActionStandard {}
class GlobalErrorClear extends ActionStandard {}

const reducerError = createReducer(undefined, {
  [GlobalErrorInit.type]: (state, action) => action.payload,
  [GlobalErrorClear.type]: (state, action) => undefined,
})
```

Then in your side-effect's `catch` block dispatch `ErrorInit`. It could look like this with [redux-thunk](https://github.com/reduxjs/redux-thunk):

```js
const catsGetAsync = async (dispatch) => {
  dispatch(new CatsGetInit())
  try {
    const res = await fetch('https://cats.com/api/v1/cats')
    const body = await res.json()
    dispatch(new CatsGetSuccess(body))
  } catch (error) {
    dispatch(new CatsGetError(error))
    dispatch(new GlobalErrorInit(error))
  }
}
```

Then you could stop providing a reducer for `error` part of cats' state and `CatsGetError` just to flip `loading` flag.

```js
class CatsGetInit extends ActionStandard {}
class CatsGetSuccess extends ActionStandard {}
class CatsGetError extends ActionStandard {}

const reducerCatsLoading = createReducer(
  false,
  reducerLoadingMap(CatsGetInit, CatsGetSuccess, CatsGetError),
)
const reducerCatsData = createReducer(undefined, {
  [CatsGetSuccess.type]: () => action.payload,
})

const reducerCats = combineReducers({
  data: reducerCatsData,
  loading: reducerCatsLoading)
})
```

## Tip 5: Stop memoizing everything

Let's take a look at a mess we have with selectors one more time.  
_I omitted `makeSelectorCatsError` because of what we discovered at the previous chapter._

```js
const makeSelectorCatsData = () =>
  createSelector(
    (state) => state.cats.data,
    (cats) => cats,
  )
const makeSelectorCatsLoading = () =>
  createSelector(
    (state) => state.cats.loading,
    (loading) => loading,
  )
```

Why would we create memoized selectors for everything? What's there to memoize? Picking an object's field by key (which is exactly what's happening here) is O(1). Just write a regular non-memoized function. Use memoization only when you want to change the shape of the data in your store in a way that requires non-constant time before returning it to your component.

```js
const selectorCatsData = (state) => state.cats.data
const selectorCatsLoading = (state) => state.cats.loading
```

Memoization could make sense only if computed some derived data. For this example let's imagine that each cat is an object with field `name` and we need a string containing names of all cats.

```js
const makeSelectorCatNames = () =>
  createSelector(
    (state) => state.cats.data,
    (cats) => cats.data.reduce((accum, { name }) => `${accum} ${name}`, ''),
  )
```

## Conclusion

Let's take a look at what we started with:

```js
import { createSelector } from 'reselect'

const actionTypeCatsGetInit = 'CATS_GET_INIT'
const actionTypeCatsGetSuccess = 'CATS_GET_SUCCESS'
const actionTypeCatsGetError = 'CATS_GET_ERROR'

const actionCatsGetInit = () => ({ type: actionTypeCatsGetInit })
const actionCatsGetSuccess = (payload) => ({
  type: actionTypeCatsGetSuccess,
  payload,
})
const actionCatsGetError = (error) => ({
  type: actionTypeCatsGetError,
  payload: error,
})

const reducerCatsInitialState = {
  error: undefined,
  data: undefined,
  loading: false,
}
const reducerCats = (state = reducerCatsInitialState, action) => {
  switch (action.type) {
    case actionTypeCatsGetInit:
      return {
        ...state,
        loading: true,
      }
    case actionCatsGetSuccess:
      return {
        error: undefined,
        data: action.payload,
        loading: false,
      }
    case actionCatsGetError:
      return {
        ...data,
        error: action.payload,
        loading: false,
      }
    default:
      return state
  }
}

const makeSelectorCatsData = () =>
  createSelector(
    (state) => state.cats.data,
    (cats) => cats,
  )
const makeSelectorCatsLoading = () =>
  createSelector(
    (state) => state.cats.loading,
    (loading) => loading,
  )
const makeSelectorCatsError = () =>
  createSelector(
    (state) => state.cats.error,
    (error) => error,
  )
```

And what the result is:

```js
class CatsGetInit extends ActionStandard {}
class CatsGetSuccess extends ActionStandard {}
class CatsGetError extends ActionStandard {}

const reducerCatsLoading = createReducer(
  false,
  reducerLoadingMap(CatsGetInit, CatsGetSuccess, CatsGetError),
)
const reducerCatsData = createReducer(undefined, {
  [CatsGetSuccess.type]: () => action.payload,
})

const reducerCats = combineReducers({
  data: reducerCatsData,
  loading: reducerCatsLoading)
})

const selectorCatsData = (state) => state.cats.data
const selectorCatsLoading = (state) => state.cats.loading
```

Hopefully, you found something useful for your project. Feel free to communicate your feedback back to me! I most certainly appreciate any criticism and questions.
