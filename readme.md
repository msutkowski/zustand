<p align="center">
  <img width="500" src="bear.png" />
</p>

![Bundle Size](https://badgen.net/bundlephobia/minzip/zustand) [![Build Status](https://travis-ci.org/react-spring/zustand.svg?branch=master)](https://travis-ci.org/react-spring/zustand) [![npm version](https://badge.fury.io/js/zustand.svg)](https://badge.fury.io/js/zustand) ![npm](https://img.shields.io/npm/dt/zustand.svg)

Small, fast and scaleable bearbones state-management solution. Has a comfy api based on hooks, isn't boilerplatey or opinionated, but still just enough to be explicit and flux-like.

Don't disregard it because it's cute. It has quite the claws, lots of time was spent to deal with common pitfalls, like the dreaded [zombi child problem](https://react-redux.js.org/api/hooks#stale-props-and-zombie-children), [react concurreny](https://github.com/bvaughn/rfcs/blob/useMutableSource/text/0000-use-mutable-source.md), and [context loss](https://github.com/facebook/react/issues/13332) between mixed renderers. It may be the one state-manager in the React space that gets all of these right.

You can try a live demo [here](https://codesandbox.io/s/dazzling-moon-itop4).

    npm install zustand

### First create a store

Your store is a hook! You can put anything in it, atomics, objects, functions. The `set` function *merges* state.

```jsx
import create from 'zustand'

const useStore = create(set => ({
  bears: 0,
  increasePopulation: () => set(state => ({ count: state.bears + 1 })),
  resetPopulation: () => set({ bears: 0 })
}))
```

### Then bind your components, and that's it!

Use the hook anywhere, no providers needed. Select your state and the component will re-render on changes.

```jsx
function BearCounter() {
  const bears = useStore(state => state.bears)
  return <h1>{bears} around here ...</h1>
}

function Controls() {
  const increasePopulation = useStore(state => state.increasePopulation)
  return <button onClick={increasePopulation}>one up</button>
}
```

#### Why zustand over react-redux?

* Simple and un-opinionated
* Makes hooks the primary means of consuming state
* Doesn't wrap your app into context providers
* Can inform components transiently (without causing render)

---

# Recipes

## Fetching everything

You can, but bear in mind that it will cause the component to update on every state change!

```jsx
const state = useStore()
```

## Selecting multiple state slices

It detects changes with strict-equality (old === new) by default, this is efficient for atomic state picks.

```jsx
const nuts = useStore(state => state.nuts)
const honey = useStore(state => state.honey)
```

If you want to construct a single object with multiple state-picks inside, similar to redux's mapStateToProps, you can tell zustand that you want the object to be diffed shallowly by passing an alternative equality function.

```jsx
import shallow from 'zustand/shallow'

// Object pick, re-renders the component when either foo or bar change
const { nuts, honey } = useStore(state => ({ nuts: state.nuts, honey: state.honey }), shallow)

// Array pick, re-renders the component when either foo or bar change
const [nuts, honey] = useStore(state => [state.nuts, state.honey], shallow)

// Mapped picks, re-renders the component when state.objects changes in order, count or keys
const treats = useStore(state => Object.keys(state.treats), shallow)
```

## Fetching from multiple stores

Since you can create as many stores as you like, forwarding results to succeeding selectors is as natural as it gets.

```jsx
const currentBear = useCredentialsStore(state => state.currentBear)
const bear = useBearStore(state => state.bears[currentBear])
```

## Memoizing selectors

It is generally recommended to memoize selectors with useCallback. This will prevent unnecessary computations each render. It also allows React to optimize performance in concurrent mode.

```jsx
const fruit = useStore(useCallback(state => state.fruits[id], [id]))
```

If a selector doesn't depend on scope, you can define it outside the render function to obtain a fixed reference without useCallback.

```jsx
const selector = state => state.berries

function Component() {
  const berries = useStore(selector)
```

## Overwriting state

The `set` function has a second argument, false by default. Instead of merging, it will replace the state model. Be careful not to wipe out parts you rely on, like actions.

```jsx
import omit from "lodash-es/omit"

const useStore = create(set => ({
  salmon: 1,
  tuna: 2,
  deleteEverything: () => set({ }), true), // clears the entire store, actions included
  deleteTuna: () => set(state => omit(state, ['tuna']), true)
}))
```

## Async actions

Just call `set` when you're ready, it doesn't care if your actions are async or not.

```jsx
const useStore = create(set => ({
  fishies: {},
  fetch: async pond => {
    const response = await fetch(pond)
    set({ fishies: await response.json() })
```

## Read from state in actions

`set` allows fn-updates `set(state => result)`, but you still have access to state outside of it through `get`.

```jsx
const useStore = create((set, get) => ({
  sound: "grunt",
  action: () => {
    const sound = get().sound
```

## Reading/writing state and reacting to changes outside of components

Sometimes you need to access state in a non-reactive way, or act upon the store. For these cases the resulting hook has utility functions attached to its prototype.

```jsx
const useStore = create(() => ({ paw: true, snout: true, fur: true }))

// Getting non-reactive fresh state
const paw = useStore.getState().paw
// Listening to all changes, fires on every change
const unsub1 = useStore.subscribe(console.log)
// Listening to selected changes, in this case when "paw" changes
const unsub2 = useStore.subscribe(console.log, state => state.paw)
// Updating state, will trigger listeners
useStore.setState({ paw: false })
// Unsubscribe listeners
unsub1()
unsub2()
// Destroying the store (removing all listeners)
useStore.destroy()

// You can of course use the hook as you always would
function Component() {
  const paw = useStore(state => state.paw)
```

## Using zustand without React

Zustands core can be imported and used without the React dependency. The only difference is that the create function does not return a hook, but the api utilities.

```jsx
import create from 'zustand/vanilla'

const store = create(() => ({ ... }))
const { getState, setState, subscribe, destroy } = store
```

You can even consume an existing vanilla store with React:

```jsx
import create from 'zustand'
import vanillaStore from './vanillaStore'

const useStore = create(vanillaStore)
```

## Transient updates (for often occuring state-changes)

The subscribe function allows components to bind to a state-portion without forcing re-render on changes. Best combine it with useEffect for automatic unsubscribe on unmount. This can make a [drastic](https://codesandbox.io/s/peaceful-johnson-txtws) performance impact when you are allowed to mutate the view directly.

```jsx
const useStore = create(set => ({ scratches: 0, ... }))

function Component() {
  // Fetch initial state
  const scratchRef = useRef(useStore.getState().scratches)
  // Connect to the store on mount, disconnect on unmount, catch state-changes in a reference
  useEffect(() => useStore.subscribe(
    scratches => (scratchRef.current = scratches), 
    state => state.scratches
  ), [])
```

## Sick of reducers and changing nested state? Use Immer!

Reducing nested structures is tiresome. Have you tried [immer](https://github.com/mweststrate/immer)?

```jsx
import produce from 'immer'

const useStore = create(set => ({
  nested: { structure: { contains: { a: "bear" } } },
  set: fn => set(produce(fn)),
}))

const set = useStore(state => state.set)
set(state => {
  state.nested.structure.contains = null
})
```

## Middleware

You can functionally compose your store any way you like.

```jsx
// Log every time state is changed
const log = config => (set, get, api) => config(args => {
  console.log("  applying", args)
  set(args)
  console.log("  new state", get())
}, get, api)

// Turn the set method into an immer proxy
const immer = config => (set, get, api) => config(fn => set(produce(fn)), get, api)

const useStore = create(log(immer(set => ({
  bees: false,
  setBees: input => set(state => {
    state.bees = input
  })
}))))
```

## Can't live without redux-like reducers and action types?

```jsx
const types = { increase: "INCREASE", decrease: "DECREASE" }

const reducer = (state, { type, by = 1 }) => {
  switch (type) {
    case types.increase: return { grumpiness: state.grumpiness + by }
    case types.decrease: return { grumpiness: state.grumpiness - by }
  }
}

const useStore = create(set => ({
  grumpiness: 0,
  dispatch: args => set(state => reducer(state, args)),
}))

const dispatch = useStore(state => state.dispatch)
dispatch({ type: types.increase, by: 2 })
```

Or, just use our redux-middleware. It wires up your main-reducer, sets initial state, and adds a dispatch function to the state itself and the vanilla api. Try [this](https://codesandbox.io/s/amazing-kepler-swxol) example.

```jsx
import { redux } from 'zustand/middleware'

const useStore = create(redux(reducer, initialState))
```

## Redux devtools

```jsx
import { devtools } from 'zustand/middleware'

// Usage with a plain action store, it will log actions as "setState"
const useStore = create(devtools(store))
// Usage with a redux store, it will log full action types
const useStore = create(devtools(redux(reducer, initialState)))
```

devtools takes the store function as its first argument, optionally you can name the store with a second argument: `devtools(store, "MyStore")`, which will be prefixed to your actions.
