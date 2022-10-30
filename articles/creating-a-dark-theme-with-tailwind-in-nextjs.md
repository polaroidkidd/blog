# Creating a Dark Theme with Tailwind in Nextjs

**Author: Daniel Einars**

**Date Published: 30.10.2022**

## 1. Intro

With nextjs becoming the gold standard for developing react applications I thought I'd briefly explain how to create a
nice dark theme using nextjs and tailwind. We're going build the following

1. Theme Toggler
2. Theme Context Provider
3. Example Usage

By the end of it you'll be able to

Prerequisites:

* A running nextjs application with tailwind configured.
* Following dependencies:
    * `js-cookie@^3.0.1` & `@types/js-cookie@^3.0.2`to persist the theme
    * `tailwindcss@3.0.5` for styling the page

## 2. Theme Context Provider

Before we start. Tailwind implements their dark theme by css child selectors. Basically, if you're `HTML` element
has `class="dark"`, them it will automatically apply all `dark:some-tailwind-class` styles. That's why all the
functionallity around this will involve adding/removing `class="dark"` when toggeling a theme.

We'll be using `createContext` in order to keep track of the theme and witch it when we want. In this instance.

> *But why are we using react's context? It causes a lot of rerenders and.. stuff*!

.. to which I say: *"nu-uh! react's context is a dependency injection tool and if we don't bastardise it's intended
usage we're not causing any harm!"*

For everyone who just wants to copy&paste everything just scroll to the bottom for the completed work.

### 2.1 Creating the Context.

This is fairly straight forward, so I won't dive into any details.

```typescript jsx

const initialState = false; // we start the first-time visitors up on the light theme

export const ThemeContext = createContext({
  isDarkTheme: initialState, // pass in the inital state
  toggleThemeHandler: () => {
  }, // define a function to toggle the theme
});
```

### 2.2 The Theme Context Provider

The Context Provider has to handle the following two cases.

1. New user:
    1. Default to light theme
    2. Set light theme cookie
    3. Set `class="light"` on the `HTML` element
2. Returning user:
    1. Read cookie
    2. Call `setIsDarkTheme` with appropriate value
    3. Set `class="light"` or `class="dark"` on the `HTML` element (technically we don't need the `light` class, but I
       like to keep it there)
3. Change the theme when the user wants to

We're going to need two functions. One to initialize the theme and handle cases `1` and `2`. And a function to actually
toggle the theme.

#### 2.2.1 Initializer

The `ThemeContextProvider` keeps track of which theme we currently have enabled in its own useState call. You could rely
on the cookie exclusively, but I found that to be a bit of a pain. Hence, we
have `const [isDarkTheme, setIsDarkTheme] = useState(initialState);` at the very top. This way we can also
call `initialThemeHandler` whenever the user decided to change the theme and we don't have to separate
a `initialThemeHandler` function from a `setTheme` function.

```typescript jsx
  const [isDarkTheme, setIsDarkTheme] = useState(initialState);

const initialThemeHandler = useCallback((): void => {
  // Get current cookie theme.
  const themeCookie = Cookies.get("theme");
  // js-cookie returns `undefined` if there's no cookie by that name
  setIsDarkTheme(themeCookie === "dark");
  // Here we start handling the case when we have a returning user (because we found a cookie)
  if (themeCookie) {
    // Just to be super-duper sure we're adding the right classes, remove what ever the other theme is
    document.querySelector("html")?.classList.remove(isDarkTheme ? "dark" : "light");
    // The cookie will have the value of `dark` or `light`
    // Therefore we can just set it to the value of the cookie
    document.querySelector("html")?.classList.add(themeCookie);
  } else {
    // Oooo!! A new user!
    // I set the cookie expiration to 30 days, but that's optional
    const date = new Date();
    const expires = new Date(date.setMonth(date.getMonth() + 1));
    // Set the default light cookie
    Cookies.set("theme", "light", {
      secure: true,
      expires: expires,
    });
  }
  // we're going to call this callback everytime the `isDarkTheme` property changes
}, [isDarkTheme]);

// an dalso on the initial render
useEffect(() => initialThemeHandler(), [initialThemeHandler]);
```

#### 2.2.2 Theme toggle function

This is the function which the context will provide to other components via (say it with me) *dependency injection*.

```typescript jsx
  function toggleThemeHandler(): void {
  // get the current theme cookie. We know it exists since this will 100% of the time run after the `initialThemeHandler` function
  const themeCookie = Cookies.get("theme");
  // What ever theme we previously had, set it to the opposite.
  // Remember this is a boolean!
  setIsDarkTheme((ps) => !ps);
  // Create a new cookie expiration date
  const date = new Date();
  const expires = new Date(date.setMonth(date.getMonth() + 1));
  // Set the cookie to the opposit of what ever it's currently holding
  Cookies.set("theme", themeCookie !== "dark" ? "dark" : "light", {
    secure: true,
    expires: expires,
  });

  // add the appropriate class to the `HTML element
  toggleDarkClassToHTMLElement();
}
```

All this function does is remove either `dark` or `light` and then add either `dark` or `light` to the `HTML` element.
I chose this element because it's the top most one and I ran into some issues with nextjs and changing the `body`
classes.

```typescript
  function toggleDarkClassToHTMLElement(): void {
  document.querySelector("html")?.classList.remove(isDarkTheme ? "dark" : "light");
  document.querySelector("html")?.classList.add(!isDarkTheme ? "dark" : "light");
}

```

Lastly we return the `ThemeContext.Provider` like this

```typescript jsx
  return (
  <ThemeContext.Provider
    value={
      {
        isDarkTheme, // remember, this is a boolean
        toggleThemeHandler // handler to toggle the theme
      }}>
    {props.children} // all other components will be child components of this one
  </ThemeContext.Provider>
);
```

Next we need to *initialize* the theme. That should handle the following scenarios

```typescript jsx
  const initialThemeHandler = useCallback((): void => {
  // Get current cookie theme.
  const themeCookie = Cookies.get("theme");
  // js-cookie returns `undefined` if there's no cookie by that name
  setIsDarkTheme(themeCookie === "dark");
  // Here we start handling the case when we have a returning user (because we found a cookie)
  if (themeCookie) {
    // Just to be super-duper sure we're adding the right classes, remove what ever the other theme is
    document.querySelector("html")?.classList.remove(isDarkTheme ? "dark" : "light");
    // The cookie will have the value of `dark` or `light`
    // Therefore we can just set it to the value of the cookie
    document.querySelector("html")?.classList.add(themeCookie);
  } else {
    // Oooo!! A new user!
    // I set the cookie expiration to 30 days, but that's optional
    const date = new Date();
    const expires = new Date(date.setMonth(date.getMonth() + 1));
    // Set the default light cookie
    Cookies.set("theme", "light", {
      secure: true,
      expires: expires,
    });
  }
  // we're going to call this callback everytime the `isDarkTheme` property changes
}, [isDarkTheme]);

// an dalso on the initial render
useEffect(() => initialThemeHandler(), [initialThemeHandler]);
```

## 3. Applying themes

Since we want *all* our components to be able to access the current theme and toggle it, we're going to wrap our entire
nextjs app in the provider. To do this we create a `_app.tsx` file and wrap all components in the provider like this

```typescript jsx
export default function Root({Component, pageProps}: AppProps) {
  return (
    <ThemeContextProvider>
      <Component {...pageProps} />
    </ThemeContextProvider>
  );
}
```

For now, we'll be using an old-fashioned button to toggle the theme.

``` typescript jsx

interface IThemeTogglerContext{
    isDarkTheme: boolean;
    toggleThemeHandler: () => void;
}

export function ThemeToggleButton(){

  // get the `toggleThemeHandler` via *dependency injection*
  const { toggleThemeHandler }: IThemeTogglerContext = useContext(ThemeContext);
  
  // toggle the theme onClick
  function toggle(){
    toggleThemeHandler()
  }
  
  // super fancy button
  return (
  <button 
    onClick
    class={classNames(
         // styles which won't be affected by the theme
        "font-bold py-2 px-4 rounded-full",
        // light theme styles
        "bg-blue-500 hover:bg-blue-700 text-white",
        // dark theme styles
        "dark:bg-blue-100 dark:hover:bg-blue-200 dark:text-red-50", 
    )}>
    Toggle Theme
  </button>)
}
```

You can then place this button anywhere you want and it'll update the theme. As mentionind in the beginning, tailwind applies the dark-theme using CSS selectors, so any styles you want in a dark theme, just prefix the selector with a `dark:` prefix, as it's d


## 4. Styling the Body and adding theme switching transition

Because I want to see some sort of small transition when I change themes, I also created a `_document.tsx` file and added some tailwind classes, which make the theme switching a pleasent experience. Here's the completed work

```typescript jsx
import Document, { Head, Html, Main, NextScript } from "next/document";

export default class _Document extends Document {
  render() {
    return (
      <Html>
        <Head>
          <title>dle.dev</title> 
        </Head>
        <body className="bg-neutral-50 dark:bg-neutral-900 transition-colors overflow-x-hidden ">
          <Main />
          <NextScript />
        </body>
      </Html>
    );
  }
}

```

*Yes, I know my `Head` / `Title` config  isn't best practice. Check out what vercel is saying on the subject [here](https://nextjs.org/docs/messages/no-title-in-document-head).*

## 5. Entire Snippit

There you go, that's all it took! In case you want to try it out, here's the complete `ThemeContext` component:

```typescript jsx

import type { ReactElement, ReactNode } from "react";
import { createContext, useCallback, useEffect, useState } from "react";
import Cookies from "js-cookie";

const initialState = false;

export const ThemeContext = createContext({
  isDarkTheme: initialState,
  toggleThemeHandler: () => {},
});

interface ThemePropsInterface {
  children: ReactNode;
}

export function ThemeContextProvider(props: ThemePropsInterface): ReactElement {
  const [isDarkTheme, setIsDarkTheme] = useState(initialState);

  const initialThemeHandler = useCallback((): void => {
    const themeCookie = Cookies.get("theme");
    setIsDarkTheme(themeCookie === "dark");
    if (themeCookie) {
      document.querySelector("html")?.classList.remove(isDarkTheme ? "dark" : "light");
      document.querySelector("html")?.classList.add(themeCookie);
    } else {
      const date = new Date();
      const expires = new Date(date.setMonth(date.getMonth() + 1));
      Cookies.set("theme", "light", {
        secure: true,
        expires: expires,
      });
    }
  }, [isDarkTheme]);
  useEffect(() => initialThemeHandler(), [initialThemeHandler]);

  function toggleThemeHandler(): void {
    const themeCookie = Cookies.get("theme");
    setIsDarkTheme((ps) => !ps);
    const date = new Date();
    const expires = new Date(date.setMonth(date.getMonth() + 1));
    Cookies.set("theme", themeCookie !== "dark" ? "dark" : "light", {
      secure: true,
      expires: expires,
    });
    toggleDarkClassToBody();
  }

  function toggleDarkClassToBody(): void {
    document.querySelector("html")?.classList.remove(isDarkTheme ? "dark" : "light");
    document.querySelector("html")?.classList.add(!isDarkTheme ? "dark" : "light");
  }

  return (
    <ThemeContext.Provider value={{ isDarkTheme, toggleThemeHandler }}>
      {props.children}
    </ThemeContext.Provider>
  );
}

```