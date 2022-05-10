[next »](02-page-skeleton.md)

## 1. Setup from scratch

There exist several ways to create a React app. 
The most common way is to use the [create-react-app](https://reactjs.org/docs/create-a-new-react-app.html) from facebook itself.
It is a perfect fit for single page apps. 
There are other use cases where a solution like [NextJs by Vercel](https://nextjs.org/learn/basics/create-nextjs-app/setup) might fit better,
e.g. the requirement of SEO with server side rendering. NextJs therefore requires Node on the production server.
In this tutorial we go with [create-react-app](https://reactjs.org/docs/create-a-new-react-app.html) because it is enough for our needs
and does not really affect the structure of our architecture in this tutorial.

[Create-react-app](https://reactjs.org/docs/create-a-new-react-app.html) is built with the [webpack](https://webpack.js.org/)
bundler which supports several [loaders](https://webpack.js.org/loaders/) which e.g. transpile newer javascript (ES6) to older (ES5)
and loaders which convert Typescript into Javascript, transpiles json files into JS and so on.
Not only webpack transpiles code it also enables hot reloading and code splitting and more with no effort.
All in all: A lot of stuff we don't need to take care about.

```typescript
const foo = 'foo';
```

TODO: mention `.gitignore`

```
.idea/
```

[next »](02-page-skeleton.md)