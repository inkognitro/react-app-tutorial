[« previous](02-authentication.md) | [next »](04-foo.md)

## 3. Routing 
Within this chapter we go through routing stuff and create the page layout base.

### 3.1 Add two more aliases
First, we should create two more aliases:
A `@components` alias for the new `src/component/` folder where our general app specific components live in,
and a `@pages` alias for the new `src/pages` folder where our page components will be defined.
The procedure is the same as described in [`2.7 Ramp up our import paths`](02-authentication.md).
I am sure you can manage to get that done on your own :sunglasses:

### 3.2 Routing to page components
Every browser app with multiple pages requires some routing mechanism.
In case of React the most common library to have in mind is [react-router](https://reactrouter.com).
We could build our own routing logic, but I think for such a common task it absolutely makes sense
to take a proven solution which goes hand in hand with the browser history.
Nice to know: [react-router](https://reactrouter.com) does also provide a router for [react-native](https://reactnative.dev).

Install it with:
```
npm install react-router-dom --save 
```

### 3.x Adding styled-components
To enable the usage of CSS in our components, let's install the [styled-components](https://styled-components.com) library.

```
npm install styled-components --save
```

We also need to install the TS types for it, because the library does not support TS by default.

```
npm install @types/styled-components --save-dev
```

[« previous](02-authentication.md) | [next »](04-foo.md)