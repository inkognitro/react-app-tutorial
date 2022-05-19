[« previous](02-authentication.md) | [next »](04-foo.md)

## 3. Routing 
Within this chapter we go through page layouts, routing stuff.
We are also going to use some material ui components.

### 3.1 Add two more aliases
First, we should create two more aliases:
A `src/component/` folder where our general app specific components live in,
and a `src/pages` folder where our page components are defined.
The procedure is the same as described in `2.7 Ramp up our import paths`
in [authentication part of this tutorial](02-authentication.md).
I think you can manage getting done that on your own :sunglasses:

### 3.2 Choose a routing library
Every browser app with multiple pages requires some routing mechanism.
In case of React the most common library to have in mind is [react-router](https://reactrouter.com).
We could build our own routing logic, but I think for such a common task it absolutely makes sense
to take a proven solution which goes hand in hand with the browser history.
Nice to know: [react-router](https://reactrouter.com) does also provide a router for [react-native](https://reactnative.dev).

Install it with:
```
npm install react-router-dom --save 
```

[« previous](02-authentication.md) | [next »](04-foo.md)