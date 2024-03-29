# React TypeScript Basics
This contents of this blog are based off of the excellent [course](https://www.scotttolinski.com/) by [Scott Tolinski](https://www.scotttolinski.com/)


**Author: Daniel Einars**

**Date Published: 07.02.2020**


### UPDATE
`React.FC` isn't really used anymore and I've removed/replaced all examples using this. 

## 1. Overview

After a quick introduction to Typescript I'll highlight some brief examples & explanations of Typescript usage with React Children, React Hooks and  HTMLElements (with and without React). 
Finally, I'll sum up the differences between `Interface` and `Type` (and when to use which) as well as showing how to use untyped libraries in Typescript projects.  

## 2. Typed Props

`type Props` can be declared as a `type` or as an `interface`. 

What should I use? According to [this article](https://medium.com/@koss_lebedev/type-aliases-vs-interfaces-in-typescript-based-react-apps-e77c9a1d5fd0)

  >  If you write object-oriented code — use interfaces, if you write functional code — use type aliases.

The main difference is

  > interfaces are more “extendable” due to the support of declaration merging, and types are more “composable” due to support of union types.

Basically, you can extend existing interfaces which might become confusing in a large code base.

Example:

```tsx
interface IUser {
  firstName: string
  lastName: string
}

interface IUser {
  age: number
}

const user: IUser = {
  firstName: 'Jon',
  lastName: 'Doe',
  age: 25,
}
```

Now to the course. This is how you declare and use a `type` in a Functional TS React Component

```tsx
import React from 'react';

// type declaration:
type Props = {
  title: string,
  isActive: boolean
}

export const Header= ({title, isActive}:Props) => {
  return (
    <>
      <h1>{title}</h1>
      {isActive && <h3> Active</h3>}
    </>
  );
};
```

## 3. Default and/or Optional Props

Props can be marked as optional using the `?`. If this is done, the component requires the prop to contain a default value.

```tsx
import React from 'react';


type Props = {
  title: string,
  isActive?: boolean // optional
}

// isActive is now optional and therefor needs a default argument in case this isn't provided.
export const Header = ({title, isActive= true}:Props) => {

[...]

}

```


## 4. Types

This video covers some common types. It does not go into specifics about how you would create these when you'd actually use the `<Header [...]/>` component.


```tsx
// User Type
type User = {
  name: string;
}
// Common types.
type Props = {
  title: string, //strings
  isActive?: boolean //booleans
  thing: number // numbers
  thing2: string[] //arrays of strings || booleans || numbers
  status: 'loading' | 'loaded' // Union Type. Accepts only one of these two strings
  thing3: object // this is an empty object. can also be declared using {}
//  thing4: User | {}  the above is typically declared like this, if at all.
  thing4: {
    name: string;
  }
  func: () => void // Define functions. Quite common
  user: User
}
```


## 5. Function Props


Way in which a function can be typed.

```tsx
import React from 'react';


type Props = {
  // onClick: Function // function (lower-case "f") is not recognized. Topic of chapter 7.
  // method returns a string
  // onClick(): string // function which returns a string.
  
  // method which does not anything.
  // onClick(value: string): void
  
  // most common way to type a function.
  onClick: (text: string) => void;
}


export const Button = ({onClick}: Props) => {
  return <button onClick={() => onClick("hi there")}>Click Me</button>;
};
```


Accompanying App.tsx. Notice that I don't pass in the value to be logged in App.tsx but in Button.tsx

```tsx
import React from 'react';
import './App.css';
import { Header } from './components/Header';
import { Button } from './components/Button';

const App = () => {

  return (
    <>
      <Header
        title="{'Hello'}"
      />
      <Button
        onClick={(text) => {
          console.log(text);
        }}
      />
    </>
  );
};

export default App;

```


## 6. React Events in TypeScript

React has its own set of Events, such as React.MouseEvent. This will accept events for all clicks (bad)
```tsx
import React from "react"

type Props = {
  onClick: (e: React.MouseEvent) => void;
}
```

Now it is specified to only accept events from a button click (good)
```tsx
import React from "react"

type Props = {
  onClick: (e: React.MouseEvent<HTMLButtonElement>) => void
}

```


## 7. Typing Children Props

The example below is receiving a `string` element as a child. 

Because we are passing a string into the button component, this will work. However, down the line it will become difficult because you might want to pass a myriad of various
components into other components.

```tsx
import React from 'react';

type Props = {
  onClick: (e: React.MouseEvent<HTMLButtonElement>) => void
  children: string;
}

// Placing the Prop declaration into the arrow handles. Use this only when children are in the mix. 
// Otherwise keep on using the previous syntax.
export const Button = ({onClick, children: Props}) => {
  return <button onClick={onClick}>{children}</button>;
};
```

In order to solve this, declare the react component with `React.FC<Props>` (see Update). Placing the Prop declaration into the arrow handles. Use this only when children are in the mix. Otherwise keep on using the previous syntax. This way it merges the Props which are expected from `React.FC` with the props we declared.

```tsx
import React from 'react';

type Props = {
  onClick: (e: React.MouseEvent<HTMLButtonElement>) => void
}

export const Button = ({onClick, children}:Props) => {
  return <button onClick={onClick}>{children}</button>;
};
```

## 8. Typing React.useState

Because the default state is set to '', typescript will automatically recognized that the `useState` instance  accepts texts values. TypeScript is doing it's job implicity.
```tsx
import React from 'react';

export const Input = () => {
  const [name, setName] = React.useState('');
  return (
    <input value="{name}" onChange={e => setName(e.target.value)}/>
  );
};
```

Using the union type brackets (<>) we can define a state to be either of type string OR null. This will still throw an error because "value" can not be null.

```tsx
import React, { useState } from 'react';

export const Input = () => {
  const [name, setName] = useState<string | null>("");
  
  return (
    <input value="{name}" onChange={e => setName(e.target.value)}/>
  );
};
```

Recommendation is to set the initial type to what the state expects. If that isn't enough, union types can be used.


## 9. Using React.useRef and typing dom elements

Type refs by declaring what type of HTML Element they will be attached to. Adding the "!" to the end of "null" declares it as a read-only value (very typical for refs)

```tsx
import React, { useRef, useState } from 'react';

export const Input = () => {
  const [name, setName] = useState('');
  const ref = useRef<HTMLInputElement>(null!);
  
  console.log(ref?.current?.value);
  
  return (
    <input ref="{ref}" value="{name}" onChange={e => setName(e.target.value)}/>
  );
};
```

## 10. Using React.useReducer

Typing `useReducer` is no different than previous standard typing. You define types for `Action` & `State` and assign these to the `initialState` and the `reducer` function. Optionally add a `payload` to the action which can be typed as anything previously covered.


```tsx
import React, { useReducer } from 'react';

const initialState: State = {rValue: true};

type State = {
  rValue: boolean
}

type Action = {
  type: string
  // Optionally define a payload of any type (object, boolean, etc.)
  // payload: string
}

const reducer = (state: State, action: Action) => {
  switch (action.type) {
    case 'one': {
      return {
        rValue: true
      };
    }
    case 'two': {
      return {
        rValue: false
      };
    }
    // Required in order to tell TS what state should be accepted in the default case.
    default: {
      return state;
    }
  }
};

export const ReducerButtons = () => {
  
  const [state, dispatch] = useReducer(reducer, initialState);
  return (
    <>
      <button onClick={() => dispatch({type: 'one'})}>Action One</button>
      <button onClick={() => dispatch({type: 'two'})}>Action Two</button>
      {state?.rValue ? <h1>visible</h1> : 'invisible'}
    </>
  );
};
```

In order to stop the reducers from accepting bad actions, you can type these as well using `UntionTypes`.

```tsx
type Action = {
  type: 'one' | 'two' // using unionTypes to declare a specific set of allowed actions.
  // Optionally define a payload of any type (object, boolean, etc.)
  // payload: string
}


// Syntax for declaring multiple actions. This can also be used to type which payloads will be accepted.
type ActionExtended =
  | { type: 'one', payload: boolean }
  | { type: 'two' }
  | { type: 'three' }
  | { type: 'four' }
  | { type: 'five' };


```

## 11. React.useEffect and Custom Hooks


This custom hook es designed to execute some code depending on weather a ref is clicked or not (close modal type).

It takes a ref to the HTML Element and the handler function. Because the handler function adds `DOM` listeners (as opposed to React listeners)
it has to be declared as standard Mouse/Touch Events (both need to be declared because we could be on a mobile device).

Additionally, because we are mutating the reference, we need to declare the `ref` as a `React.MutableRefObject<HTMLElement>`. 
```tsx

import React, { useEffect } from 'react';


const useClickOutside = (
  ref: React.MutableRefObject<HTMLElement>,
  handler: (event: MouseEvent | TouchEvent) => void) => {
  useEffect(() => {
    const listener = (event: MouseEvent | TouchEvent) => {
      if (!ref.current || ref.current.contains(event.target as Node)) {
        return;
      }
      handler(event);
    };
    
    document.addEventListener('mousedown', listener);
    document.addEventListener('touchstart', listener );
    return () => {
      document.removeEventListener('mousedown', listener);
      document.removeEventListener('touchstart', listener);
      
    };
  }, [handler, ref]);
};

export { useClickOutside };
```


## 12. Generics

By example of the `useClickOutside` hook. Before we declared its `ref` type as `ref: React.MutableRefObject<HTMLDivElement>`. This did work, but did not allow us to use it on elements other than `<div/>`. Because `HTMLDivElement` extends `HTMLElement`, we can declare the `ref` in the hook as `ref: React.MutableRefObject<HTMLElement>`. When it's actually used, the ref can contain any `HTMLElement` and TS will not throw any errors.


## 13. React.useContext

TS implicitly types the declared context without having to type it. Below is an example implementation.

```tsx
import { createContext } from 'react';

export const initialValues = {
  rValue: true
};

export const GlobalContext = createContext(initialValues);
```

We can use this example in `ReducerButtons.tsx` like this.
```tsx
import { useContext } from "react";
import { GlobalContext } from './GlobalState';

const {rValue} = useContext(GlobalContext);
```

Attempts to destructor properties, which don't exist, will throw TS errors.

```tsx
import { useContext } from "react";
import { GlobalContext } from './GlobalState';

const {does-not-exist} = useContext(GlobalContext);
```

This works just great for most cases. We can also type the context should we want to.

```tsx
import { createContext } from 'react';

type InitialValues = {
  rValue: boolean
}

export const initialValues: InitialValues = {
  rValue: true
};

export const GlobalContext = createContext(initialValues);
```


Next we update the `GlobalState` component by creating a `GlobalProvider`. This saves us from 
having to import `initialValues` where ever we use it and can instead just wrap the `GlobalProvider` component.
Additionally we refactored `GlobalState.tsx` to include the `useReducer` function from the `ReducerButtons.tsx` 
component.

The refactored `GlobalState.tsx` component looks like this now

```tsx
import React, { createContext, useReducer } from 'react';


export const initialValues = {
  rValue: true,
  // This creates an implied context without having to explicitly type it.
  // It allows us to assign the dispatch functions (turnOn & turnOff) in the `value` prop
  // passed in the `<GlobalContext.Provider ...` component
  turnOn: () => {},
  turnOff: () => {}
};

export const GlobalContext = createContext(initialValues);

type State = {
  rValue: boolean
}

type Action = {
  type: 'on' | 'off'
}
const reducer = (state: State, action: Action) => {
  switch (action.type) {
    case 'on': {
      return {
        rValue: true
      };
    }
    case 'off': {
      return {
        rValue: false
      };
    }
    default: {
      return state;
    }
  }
};


export const GlobalProvider = ({children}) => {
  const [state, dispatch] = useReducer(reducer, initialValues);
  
  return (
    <GlobalContext.Provider value="{{"
      rValue: state.rValue,
      // Functions have been added to the initialValues object. These are implicitly typed. 
      turnOn: () => dispatch({type: 'on'}),
      turnOff: () => dispatch({type: 'off'})
    }}>
      {children}
    </GlobalContext.Provider>
  );
};
``` 

The `ReducerButtons.tsx` component has been updated to work with `GlobalState.tsx`.

```tsx
import React, { useContext, useRef } from 'react';
import { useClickOutside } from './useClickOutside';
import { GlobalContext } from './GlobalState';


export const ReducerButtons = () => {
  const ref = useRef<HTMLDivElement>(null!);
  const context = useContext(GlobalContext);
  const {rValue, turnOn, turnOff} = context; // Functions extracted from the GlobalContext 
  
  
  useClickOutside(ref, () => {
    console.log('Clicked Outside'); // handler passed into useClickOutside
  });
  return (
    <div ref="{ref}">
      // These functions are implicitly typed by the initialValues object in GlobalState.tsx
      <button onClick={turnOn}>Action One</button> 
      <button onClick={turnOff}>Action Two</button>
      {rValue ? <h1>visible</h1> : <h1>invisible</h1>}
    </div>
  );
};
```


## 14. Class Based Components

Classes and FC are quite similar. The following example illustrates a simple example.

`type` and `state` definitions are placed inside generic braces (`<` and `>`).

```tsx
import React, { Component } from 'react';

type State = {
  shouldRender: boolean
}


type Props = {
  title: string;
}

class BigC extends Component<Props, State> {
  render() {
    return (
      <div>
        <h1>I'm in a class component</h1>
      </div>
    );
  }
}

export default BigC;
```

## 15. Interfaces Types

The gist from several articles is
 > When you come from OOP, use `interface`, if you're into functional programming, go for `type`


Pros for Types:
  * More constrained
  * Better for intersecting
  * Better in error messages
  * Can be used in unions
 
Pros for Interfaces:
  * Better when authoring a library
  * Extendable
  * Can be augmented


[Here](https://www.educba.com/typescript-type-vs-interface/) are some helpful tips and tricks to help you decide. In the end, what ever you choose, ***stick to it!***
 
## 16. Using untyped libraries
 
Some libraries you'd like touse in your projects don't provide their own types. In this scenario you have two options:
 
  1. Declare the types yourself in a `*.d.ts` file
  2. Install the types (usually `@types/LIBRARYMANE`) from the [DefinitelyTyped](https://github.com/DefinitelyTyped/DefinitelyTyped) repository. 

An example `*.d.ts` file can look as simple as this:
 
```tsx
declare module 'styled-components' // This can allow you to use third-party un-typed libs.
```

