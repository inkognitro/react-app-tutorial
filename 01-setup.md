[« introduction](README.md) | [next »](02-authentication.md)

## 1. Setup from scratch

There exist several ways to create a React app. 
The most common way is to use the [create-react-app](https://reactjs.org/docs/create-a-new-react-app.html) from facebook itself.
It is a perfect fit for single page apps. 
There are other use cases where a solution like [NextJs by Vercel](https://nextjs.org/learn/basics/create-nextjs-app/setup) might fit better,
e.g. the requirement of SEO with server side rendering. NextJs requires Node on the production server, create-react-app does not.
In this tutorial we go with [create-react-app](https://reactjs.org/docs/create-a-new-react-app.html) because it fits our needs.

[Create-react-app](https://reactjs.org/docs/create-a-new-react-app.html) is built with the [webpack](https://webpack.js.org/)
bundler which supports several [loaders](https://webpack.js.org/loaders/).
Webpack loaders do transpile newer javascript or even typescript to older javascript (e.g. ES5)
which is readable for all browsers.
Not only Webpack handles the transpiling, but also supports hot reloading, code splitting and finally the bundling of your
production app. So Webpack takes care of a lot of stuff which makes our life much easier.

### 1.1 Installation
With [create-react-app](https://create-react-app.dev) it goes stunningly fast to start a new React project.
The only thing you need todo is to open your console and create a new project folder by executing the command below.
This command is going to download the typescript starter template for new react apps.

```console
npx create-react-app my-project --template typescript
```
This one might take a few minutes.

As soon as the installation process is finished, you can test if everything works as expected by starting the app.
To start the app you could open the `package.json` and execute the `start` script by clicking the green "play" button.
Alternatively you can execute the script with `npm start` directly from your console (`cd my-project`, `npm start`).

Now you should be able to see the following content in your browser at `localhost:3000`:

![CreateReactApp :: Start Page](docs/01-01-start-page.png "CreateReactApp :: Start Page")

Yay! You've just started the app in the dev mode.
Your code changes are going to be hot reloaded in the browser.
Awesome! No need to refresh the page after saving your files!
Thanks to webpack and create-react-app.

### 1.2 Time to commit, but...
...before we make our first commit, let's add an entry to the recently created `.gitignore` file to prevent committing our
IDE files. Add a new line `.idea/` in case of a JetBrains IDE (e.g. WebStorm or PhpStorm) or
`.vscode/` if you use Visual Studio Code.

### 1.3 The SFC pattern
Before we start building things, I suggest cleaning up the app template a bit.

If we look at our project we can see an `src/App.css` and an `src/index.css` file.
Let's delete those imports in `src/index.tsx` and `src/App.tsx` and also the css files.
We are going to solve styling issues with [styled-components](https://styled-components.com/) and here is the **why**:

If we have look at [VueJs](https://vuejs.org), there exists a pattern called [Single File Components](https://vuejs.org/guide/scaling-up/sfc.html):

> In modern UI development, we have found that instead of dividing the codebase into three huge layers
> that interweave with one another, it makes much more sense to divide them into loosely-coupled components
> and compose them.
> Inside a component, its template, logic, and styles are inherently coupled, and collocating them actually
> makes the component more cohesive and maintainable.
>
> `https://vuejs.org/guide/scaling-up/sfc.html, 2022-05-16`

As you can see, this pattern significantly reduces code complexity.

### 1.4 Setup linting
With unified linting rules a team can agree on the same coding style.
This increases the code readability for each individual team member and therefore increases productivity.

Luckily create-react-app comes with installed linting packages out of the box.
I suggest integrating an automatic lint fixing process in your CI pipeline.
If you don't like CI processes to push changes into your codebase,
I suggest integrating at least the check of the linting rules.
To be able to automatically format your code, let's add `prettier`
which goes hand in hand with the already installed `eslint`:

```
npm install eslint-plugin-prettier@latest --save-dev
```

Just extend the `eslintConfig` in your `package.json` like so:
```json
"eslintConfig": {
    "extends": [
      "react-app",
      "react-app/jest"
    ],
    "plugins": [
      "@typescript-eslint",
      "prettier"
    ],
    "rules": {
      "prettier/prettier": "error",
      "@typescript-eslint/no-redeclare": [
        "error"
      ]
    }
},
```
Let's adjust our prettier settings a bit by adding a `.prettierrc.js` file:
```js
// .prettierrc.js

module.exports = {
    arrowParens: 'always',
    bracketSpacing: true,
    bracketSameLine: true,
    jsxSingleQuote: false,
    printWidth: 120,
    quoteProps: 'as-needed',
    rangeStart: 0,
    rangeEnd: Infinity,
    semi: true,
    singleQuote: true,
    tabWidth: 4,
    trailingComma: 'es5',
    useTabs: false,
    endOfLine: "lf",
};
```

Finally, we need to extend the `scripts` section in the `package.json` to easily access
the mentioned two options described above.

```json
"lint": "eslint ./src --ext .ts,.tsx",
"lint:fix": "tsc && eslint ./src --ext .ts,.tsx --quiet --fix"
```

You should now be able to test the linting by running `npm run lint` in your console.
This prints the result of what needs to be corrected if some code does not correspond with the linting rules.
If you run `npm run lint:fix`, the code style is going to be automatically fixed when possible.

> :bulb: To make your life easier, make sure your IDE applies the lint fixes after file savings.

Did you notice the `tsc &&` at the start of the `lint:fix` script?
This is to prevent from automatic lint fixes when type errors still exist.
Once, I forgot to implement this check, and I did run the `lint:fix` script.
The result was, that the script totally messed up my code base.
Luckily I hadn't committed yet. With this check you are save, or at least a bit more :innocent:

> :bulb: As a Windows user you could face errors in your IDE after you did a `git commit`.
> This is because Git then automatically replaces `CRLF` (Windows) with `LF` (Unix).
> You can fix that by running `lint:fix` after every commit but this would be really annoying.
> It's easier to change the end-of-line style of your code editor to `LF`
> (e.g. [see WebStorm docs](https://www.jetbrains.com/help/webstorm/configuring-line-endings-and-line-separators.html))
> and to adjust the Git configuration `core.autocrlf=false`
> (see [this article](https://stackoverflow.com/questions/1967370/git-replacing-lf-with-crlf)).

:floppy_disk: [branch 01-setup](https://github.com/inkognitro/react-app-tutorial-code/tree/01-setup)

[« introduction](README.md) | [next »](02-authentication.md)