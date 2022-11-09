# Debounce vs Throttle with examples

**Author: Daniel Einars**

**Date Published: 01.11.2022**

## 1. Overview

Every single time I'm calling a function a lot of times in quick succession I ask myself

> Wait... should I debounce call or throttle this call?

So for the sake of preserving my sanity (and maybe yours), I'll briefly explain what each function does and when to use
which.


## 2. Debounce

This function essentially cancels running calls until it hasn't been called for a set period of time. This makes it ideal for making sure that search query is only sent to the endpoint once the user is finished typing the query.

```typescript
function debounce(callback, delay = 500){
  let timeoutId;
  function debounced(...args){
    if(timeoutId){
      clearTimeout(timeoutId)
    }
    timeoutId = setTimeout(() => {
      callback(...args)
    }, delay)
  }
  return debounced
}
```

## 3. Throttle

Throttling a function call will reduce the number of times that function is called. It does that by setting a timeout
and only calling the last function once the timeout is complete. An implementation of throttle could look something like
this

```typescript
function throttle(callback, delay = 500) {
  var wait = false;

  function throttled(...args) {
    if (!wait) {
      callback(...args);
      wait = true;
      setTimeout(function () {
        wait = false;
      }, delay);
    }
  }
  
  return throttled
}

```

This snippet will cancel all calls which happen within the `delay`. Sometimes you'll want to que the last call. 

```typescript

function throttle(callback, delay = 500) {
  let shouldWait = false
  let quedArguments

  function quedCallback() {
    if (quedArguments == null) {
      shouldWait = false
    } else {
      callback(...quedArguments)
      quedArguments = null
      setTimeout(quedCallback, delay)
    }
  }


  function throttled(...args) {
    if (shouldWait) {
      quedArguments = args
      return
    }

    shouldWait = true
    setTimeout(quedCallback, delay)
  }

  return throttled
}

```

The major difference here is that we're saving the `...args` in the `quedArguments` variable and calling the `callback` with the last `quedArguments` contents.


