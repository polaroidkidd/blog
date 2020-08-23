# Advanced React Patterns V2
The contents of this article/summary are based off of the excelent course [Advanced React Patterns](https://frontendmasters.com/courses/advanced-react-patterns/) by Kent C. Dodds.


**Author: Daniel Einars**

**Date Published: 26.08.2020**

**Date Edited: 26.08.2020**


##  1. Intro
This chapter serves as a small warm-up

###  1.1. Basic Toggle Component

When updating state in a component, which references its past state, always use an updater function. React sometimes puts state changes in batches, and without the updater function you cannot guarantee what the state currently is.


```typescript jsx
state = {on: falase}

this.setState((currentState) => {!currentState.on})

```
Additionally, provide a `changeHandler` function to when updating the state, like so:

```typescript jsx

this.setState((currentState) => {!currentState.on}, () => {
    console.log("I have changed my state to: ", !currentState.on)
})
```

The `changeHandler` function can be passed in by the parent component and notify it of state changes. It's basically a funciton which runs every time the state has been updated.


#### What I learnt:

  1. Always use an updater function
  2. I can provide change handlers to state update functions through props


##  2. Compound Components

Compound components have two characteristics:

  1. They share state with their parent component
  2. They are "useless" by themselves.

Compound components are similar to html select elements:

```html
<select name="cars" id="cars">
  <option value="volvo">Volvo</option>
  <option value="saab">Saab</option>
  <option value="mercedes">Mercedes</option>
  <option value="audi">Audi</option>
</select>
```
The `select` element can be used on its own, but it really only becomes useful when used in conjunction with the `option` elements.

###  2.1. React Compound Component: Basic
Compound components can be defined in a similar fashion. The `static` instances can be iterated over using `React.Children.map` or `React.Children.forEach`. In this instance we use the `map` function because we want to return elements (as it is being done in the `render()` function) using the `React.cloneElement` function. 

The `React.cloneElement` function creates a copy of the passed in component (in this case `static On`, `static Off` or `static Button`) and passing additional props to them like so:

```typescript jsx
[...]
      return React.cloneElement(childElement, {
        on: this.state.on,
        toggle: this.toggle
      })
[...]
```

Here is the complete example.

```typescript jsx
import React from "react";

class Toggle extends React.Component {

  static On = (props) => props.on ? props.children : null;
  static Off = (props) => props.on ? null: props.children;
  static Button = ({on, toggle}) => <Switch on={on} onClick={toggle}/>

  state = {on: false}
  toggle = () =>
    this.setState(
      ({on}) => ({on: !on}),
      () => this.props.onToggle(this.state.on),
    )

  render() {

    return React.Children.map(this.props.children, childElement => {
      return React.cloneElement(childElement, {
        on: this.state.on,
        toggle: this.toggle
      })
    })
  }
}

```

#### What I learnt:

  1. I can create iterate over child components of a component using `React.Children.map` or `React.Children.forEach`
  2. I can clone React elements using `React.cloneElement` and pass in new props (by passing them in).
  3. This method stops me from having to implement conditional rendering (such as ` this.props.renderMessage ? this.props.renderMessage : undefined`)
  
###  2.2. React Compound Component: Flexible

The drawback of the above implementation is that it can not handle undefined elements, such as wrapping one of the Compounds in a `div` tag.

``` tsx
function Usage({
  onToggle = (...args) => console.log('onToggle', ...args),
}) {
  return (
    <Toggle onToggle={onToggle}>
      <Toggle.On>The button is on</Toggle.On>
      <Toggle.Off>The button is off</Toggle.Off>
      <div>
        <Toggle.Button />
      </div>
    </Toggle>
  )
}
```

To deal with this, we provide use the `React.Context` api to only give props to those children which require them. First we need to create a context. 

```jsx
const ToggleContext = React.createContext({
  on: false,
  toggle: () => {},
})
```

The object passed into the `React.createContext` function is the default state. This state is consumed by the children. In order for them to be able to consume it, we need to provide it it the `render()` function of the `Toggle` class.

```jsx
  [...]  
  render() {
    return (
      <ToggleContext.Provider value={this.state}>
        {this.props.children}
      </ToggleContext.Provider>
    )
  }
```

The `value` here is mapped to the component state.

```jsx
  [...]
  toggle = () =>
    this.setState(
      ({on}) => ({on: !on}),
      () => this.props.onToggle(this.state.on),
    )
  state = {on: false, toggle: this.toggle}
```

We use the state because we want to hinder unnecessary rendering (react only renders components if their internal state has changed). It seems a bit odd at first, but we need to pass the `toggle` function into the state, so the children of the component can "consume" this function. We destructure the provided `on` in the consumer function, (we could also provide `contextValue` and check for `contextValue.on`) and either render the children, or don't.
 ```jsx
  [...]   
  static On = ({children}) => (
    <ToggleContext.Consumer>
      {({on}) => (on ? children : null)}
    </ToggleContext.Consumer>
  )
```
 
 The `Button` compound doesn't use the components toggle function, but consumes it from the provided context.
 ```jsx
  [...]
  static Button = (props) => (
    <ToggleContext.Consumer>
      {({on, toggle}) => (
        <Switch on={on} onClick={toggle} {...props} />
      )}
    </ToggleContext.Consumer>
  )
```

#### What I learnt:
  1. That I can use `React.Context` in small components as well as in large applications. It remains isolated to this component.
  2. How I can provide shared state to specific compound components.

##  3. Render Props

This chapter deals with render props and answers which problem they solve and how to use them.

###  3.1. Render Props: Basic

The idea behind render props is to give the user implementing the responsibillity and freedom to configure how the component renders. 
This way the user has the possibillity to add additional functionallity to the component without having to re-implement anything. 

##### Example Render Prop Component:
```jsx
class Toggle extends React.Component {
  state = {on: false}
  toggle = () =>
    this.setState(
      ({on}) => ({on: !on}),
      () => {
        this.props.onToggle(this.state.on)
      },
    )
  render() {
    const {on} = this.state
    return this.props.children({on: on, toggle: this.toggle})
  }

```

Using the toggle component has requirements. Here we access the Toggle Components `on` and `toggle` function to define how the component is wired up.

###### Example Usage render prop component
```jsx
function Usage({
  onToggle = (...args) => console.log('onToggle', ...args),
}) {
  return (
    <Toggle onToggle={onToggle}>
      {({on, toggle}) => ( 
        <div>
          {on ? 'The button is on' : 'The button is off'}
          <Switch on={on} onClick={toggle} />
          <hr />
          <button aria-label="custom-button" onClick={toggle}>
            {on ? 'on' : 'off'}
          </button>
        </div>
      )}
    </Toggle>
  )
}

```


#### What I learnt:
  1. I can access functions and state from the implementing component of the render prop component.
  2. I *want* to render every state change.


###  3.2. Render Props: Prop Collections

When the responsibility of rendering shifts from the library to the user, the use is in charge of applying the correct props to the render prop components. In order to do this, it is helpful to provide `prop collection functions`. These functions retrieve the props, which are necessary for the components using render props to work. In the `Toggle` class we now provide a `togglerProps` object, which returns all the props reqired to use the `Toggle` class functionallity. 
```jsx
class Toggle extends React.Component {
  state = {on: false}
  toggle = () =>
    this.setState(
      ({on}) => ({on: !on}),
      () => this.props.onToggle(this.state.on),
    )
  getStateAndHelpers() {
    return {
      on: this.state.on,
      toggle: this.toggle,
      togglerProps: {
        'aria-pressed': this.state.on,
        onClick: this.toggle,
      },
    }
  }
  render() {
    return this.props.children(this.getStateAndHelpers())
  }
}
```

These `togglerProps` are passed into the `props.children` method, which means that they are accessible to any component implementing the `Toggle` class. We can then use the pass the props to our components as needed.

```jsx
function Usage({
  onToggle = (...args) => console.log('onToggle', ...args),
}) {
  return (
    <Toggle onToggle={onToggle}>
      {({on, togglerProps}) => (
        <div>
          <Switch on={on} {...togglerProps} />
          <hr />
          <button aria-label="custom-button" {...togglerProps}>
            {on ? 'on' : 'off'}
          </button>
        </div>
      )}
    </Toggle>
  )
}

```

#### What I learnt:
  1. I can provide functionallity to users implementing my code in wrapped functions. They do not have to call every prop explicity. 


###  3.3. Render Props: Prop Getters

The problem with prop collections on their own is that you are not allowed to overwrite any props which are in the collection. In the above example the `togglerProps` object defines an `onClick` function, which is applied to the button via `...togglerProps`. If I want to define a custom onClick function (for example if I wanted to track clicks), I would have to define this `onClick` function explicity on the button. However, this overwrites the `onClick` function provided by the `togglerProps` object as shown behlow.

```jsx
  <button 
    aria-label="custom-button"
    onClick={() => console.log("I've bene clicked!")} 
    {...togglerProps}>
    {on ? 'on' : 'off'}
  </button>
```

A possible fix for ths could be passing the `togglerProps` arguments to the new `onClick` function.
```jsx
  <button 
    aria-label="custom-button"
    {...togglerProps}>
    onClick={(...args) => {
                togglerProps.onClick(...args) // fowrading all arguments, whatever they are
                console.log("I've bene clicked!")}
            } 
    {on ? 'on' : 'off'}
  </button>
```

However, becomes cumbersome to implement at scale. In order to solve this we provide an `getTogglerProps` function, which we can pass our custom arguments to and let the render prop component handle what to do with them. For this to work we define the `getTogglerProps` function on the `Toggle` class. It accepts an object which will be the users custom attributes (for instance, the custom onClick function). The attributes which the render component cares about are extrated out (in this case, just the `onClick` function). The remaining attributes are spread out over the button.

```jsx
class Toggle extends React.Component {
  state = {on: false}
  toggle = () =>
    this.setState(
      ({on}) => ({on: !on}),
      () => this.props.onToggle(this.state.on),
    )

  getStateAndHelpers() {
    return {
      on: this.state.on,
      toggle: this.toggle,
      getTogglerProps: ({onClick, ...customProps}) => {
        return {
          onClick: (...args) => {
            onClick && onClick(...args) //If `onClick` is passed as an argument, run it as well as our own toggle function.
            this.toggle()
          },
          'aria-pressed': this.state.on,
          ...customProps, //spread the rest of the custom attributes out over the button.
        }
      },
    }
  }

  render() {
    return this.props.children(this.getStateAndHelpers())
  }
}
```

If you don't want to have to call each function individually, Kent provides a small utility function called `callAll`

```jsx
const callAll = (...fns) => (...args) => fns.forEach(fn => fn && fn(...args))
```

It accepts any number of functions, which returns a function which accepts any number of arguments and then loops over each function and if it exists, calls it with the arguments. So, instead of this
```jsx
[...]
 ({onClick, ...customProps}) => {
    return {
      onClick: (...args) => {
        onClick && onClick(...args) //If `onClick` is passed as an argument, run it as well as our own toggle function.
        this.toggle()
      },
      'aria-pressed': this.state.on,
      ...customProps, //spread the rest of the custom attributes out over the button.
    }
  },
[...]
```
you can have this (order does not matter):

```jsx
const callAll = (...fns) => (...args) => fns.forEach(fn => fn && fn(...args))

[...]
 ({onClick, ...customProps}) => {
    return {
      onClick: callAll(this.toggle, onClick),
      'aria-pressed': this.state.on,
      ...customProps, //spread the rest of the custom attributes out over the button.
    }
  },
[...]
```
#### What I learnt:
  1. I learnt how to effectively provide functionality to components without loosing the ability to pass custom attributes to components,which makes writing reusable components a lot easier. 



##  4. Controlling State

This chapter deals with controlling state. 

###  4.1. Controlling State: State Initializers

When running applications you want to be able to control the initial state as well as be able to reset the state to the initial state. This exersize asks the user to allow for a passed initial state, adds a default initial state and adds a reset function, which resets the state to the passed initial state or the default state.

#### What I learnt:
  1. Be explicit about the initial state. Ideally have it in a separate object. When you change that `initialState` object, you won't have to worry about changing the initial state in a buch of other places.
 
###  4.2. Controlling State: State Reducer

  > If a render prop enables users to control how things are rendered, state reducers enable users to control how the logic works. (Kent C. Dodds)

This part deals with allowing the user to pass his own state management mechanism into the component. Keeping in fashion with the last examples, the added state reducer, which is passed into the `Toggle` class, looks like this. It allows the user to toggle the button four times. After that it only accepts changes to the state except `on`.

```js
[...]
  toggleStateReducer = (state, changes) => {
    if (this.state.timesClicked >= 4) {
      return {...changes, on: false}
    }
    return changes
  }
[...]
```

In order to allow the the `Toggle` class to accept a state management prop, we define our own `internalSetState` function like this and replace all existing `this.setState` calls with `this.internalSetState` (still passing in either the new state object, or the function which changes the state)

```js
[...]
internalSetState = (changes, callBack) => {
    this.setState(currentState => {
      return [changes]
        .map(c => typeof c === 'function' ? c(currentState) : c)
        .map(c => this.props.stateReducer(currentState, c) || {})
        .map(c => Object.keys(c).length ? c : null)[0]
    }, callBack)
}
[...]
```

There are a couple of things going on here, so let me go into some detail. Like the `this.setState` function, the `internalSetState` function also accepts a changes object or function and a callback (to be executed after the state changes have propagated).

  1. `this.setState(currentState => {`... : Because we still want to manage state, we call the `this.setState` function but request our current state with it.
  2.  `return [changes]` : we wrap the new changes in an array so we can use `map` to iterate over the changes
  3. `.map(c => typeof c === 'function' ? c(currentState) : c)`: We check if the passed changes are a function or an object (as `this.setState accepts both, we also need to accept both) and retrieve the state by either returning it directly or running the passed changes function.
  4. `.map(c => this.props.stateReducer(currentState, c) || {})`: We call the stateReducer provided by the user implementing our `Toggle` class
  5. `.map(c => Object.keys(c).length ? c : null)[0]`: In th event that `this.props.stateReducer(currentState, c)` returned an empty object we return null in order to prevent unnecessary rerenders. Lastly we grab the first item in the array with `[0]`, which is our state object.  
  6. `}, callBack)`: We run the callback.



(example replaced `this.setState` function)

```js
[...]
    this.internalSetState(
      ({on}) => ({on: !on}),
      () => this.props.onToggle(this.state.on),
    )
[...]
```


#### What I learnt:
  1. I've learnt how to guard against empty objects in state updates (`Object.keys(object).length ? ...`)
  2. I can now write components which could either manage their own state, or allow the user to manage their state, by writing a custom `internalSetState` function, which accept external state reducer functions. Writing state based components in this way makes them more reusable, especially when combining this with the render prop pattern.


###  4.3. Controlling State: State Reducers with Change Types

In this section we add a `type` to the toggle and reset button (`default`, `reset` `forced`) to allow the user to specify more detailed behavior. The usage is updated by including a `force update` button.

```tsx
[...]
 <button onClick={() => toggle({type: 'forced'})}>
    Force Toggle
</button>
[...]
```

To handle this type of behavior the `toggleReducer` needs to be updated as well to include the `forced` action.

```tsx
[...]
toggleStateReducer = (state, changes) => {
    if (changes.type === 'forced') {
      return changes
    }
    if (this.state.timesClicked >= 4) {
      return {...changes, on: false}
    }
    return changes
}
[...]
```

###  4.4. Controlling State: Control Props

This section elaborates on how to facilitate state management through props. This allows to determine a component's behavior through props and/or through state. 
If you want to be able to do both, you have to check if a state update is coming in through props or if it's part of the state. This can easily be done by checking if the prop  in question is `undefined`.

```ts
  isControlled = (prop) => this.props[prop] !== undefined
```

If a state update comes in through the props, it stands to reason that it ***exists in the props*** (otherwise it must come from the component itself either through User Input or some other action). 

Additionally you then need to merge the external and internal state. Such a function could look like this:

```ts
  getState = (state) => {
    return Object.entries(state).reduce(
      (mergedState, [key, value]) => {
        if (this.isControlled(key)) {
          mergedState[key] = this.props[key]
        } else {
          mergedState[key] = value
        }
        return mergedState
      },
      {},
    )
  }
```

In the example by Kent, he worked the code required to use an external reducer as well. In essence it filters and returns changes, which are not controlled (external input), but still calls the `onStateChange` prop with all changes.

#### What I learnt:
  1. Clever uses for `Object`
  2. How to merge internal and external states.
  
##  5. Provider Pattern

This brief chapter will focus on solving prop drilling through usage of the `Context API` and *Higher Order Components*.

###  5.1. Provider Pattern: Context API

This brief example shows how to use the context API. it is largly similar to compund components, with the exception that a pure context API implementation breaks render props. In order to do that we have to check if we're passing standard react "children" or actual functions. 

```tsx
render() {
const ui = typeof this.props.children === 'function' ? this.props.children(this.state) : this.props.children
return (
  <ToggleContext.Provider value={this.state}>
    {ui}
  </ToggleContext.Provider>
)
}
```

#### What I learnt:
  1. I have (re)-learnt that I can pass arguments to `this.props.children` and alter the child behavior with this.


### 5.2. Provider Pattern: Higher Order Components

Higher Order Components allow its user to share code. They accept a `Component`, add features to the component and return it. Typical implementations of these are found in react-redux, react-router, etc.
The `withToggle` function accepts a `React.Component`, applies the `Toggle.Consumer` logic from previous examples and returns the wrapped component. We have to ensure that props from the Compnent are passed to the wrapped component (done via spreading) as well as forwarding any `React.ref`s which might have been applied. Lastly, we ensure that any static propertis of the passed component are ***hoisted*** onto the wrapped component (using a library), otherwise these would be lost in the wrapped component.
```tsx
function withToggle(Component) {
  const Wrapper = (props, ref) => {
    return (
      <Toggle.Consumer>
        {toggleUtils => <Component toggle={toggleUtils} ref={ref} {...props}/>}
      </Toggle.Consumer>
    )
  }
  Wrapper.displayName = `withToggle(${Component.displayName || Component.name})`
  return hoistNonReactStatics(React.forwardRef(Wrapper), Component)
}
```

We can now easily create components which use the `Toggle.Consumr` without having to explicitly type it out in every component.

#### What I learnt:
  1. That I have to pass Refs along to the wrapped component
  2. Hoisting the Static properties used to be cumbersome, but now there's a library for that :) (yay bundle size).
