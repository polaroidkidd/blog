# The Only Rounding Function You Will Ever Need

**Author: Daniel Einars**

**Date Published: 08.11.2022**


We do have calls like `Math.round` or `Math.floor`, but those round to full numbers. The first rounds to a full number and the second simply rounds down. What do you do if you need to specify to how many decimal places, or specifically, to which step you want to round?. Well, look no further. I present **drumroll** _The Last Rounding Funciton You Will Ever need_

```javascript
function round(value, step) {
    step || (step = 1.0);
    const inv = 1.0 / step;
    return Math.round(value * inv) / inv;
}
```

courtesy of [Michael Deal](https://stackoverflow.com/a/34591063).

I supposed you could improve it by also employing [EPSILON](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number/EPSILON), but I find that this suffices for the time being.