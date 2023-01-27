> :raised_hands: Special thanks go to [easylearn schweiz ag](https://easylearn.ch).
> This is the company I currently work for.
> Our managers allowed me to create parts of this tutorial at working time.

# react-app-tutorial
The purpose of this tutorial is to serve for a better understanding of the concepts of React
while getting familiar with some state-of-the-art libraries used with it in 2022.

To go through this tutorial without lots of pain, you should be familiar with Javascript ES6+, React and JSX.
This tutorial is written in Typescript.
I think most people who understand Javascript can arrange with TS by doing this tutorial.
No need to panic: It is just javascript with type hints.

### Approach
Together we are going to build a single page app with an `index` and a `register user page`.
The user registration form is going to be wired with a mocked api over http to also cover network traffic.
The output of this tutorial is a clean and reusable framework built on top of React.

### Out of scope
We do not cover [redux](https://redux.js.org/) in this tutorial.
In my opinion and especially in this time (with React 16+) I think redux is an overkill for most cases.
The trade-off between redux's code-splitting and therefore its higher complexity overwhelms the benefits of
performance optimizations. I experienced that encapsulated components with its callbacks are a lot easier to understand and
less error prone than reducers, actions and components which are split into different parts in the code base.
For global and non steadily changed state I would suggest using the React's `useContext` hook.

### Tutorial steps
Below you can see an overview of all steps of this tutorial.

1. [Setup from scratch](01-setup.md) - Installation: create-react-app, linting
2. [Authentication](02-authentication.md) - Authentication context for the current user, import paths
3. [Routing](03-routing.md) - Preparing page layouts and the page's skeletons, mui, styled-components
4. [Internationalization](04-i18n.md) - Translating contents, localization base, switching between two languages
5. [Toaster](05-toaster.md) - Providing a way to trigger messages (toasts) to the website user
6. [Form](06-form.md) - Providing basic form elements and the possibility of error enrichment
7. [Collections](07-collections.md) - Designing an interface for collection providers, implementation for arrays
8. [ApiV1](08-apiv1.md) - Standardized http request handling, provide the first API version 1 endpoint
9. [Wiring](09-wiring.md) - Wiring the registration form with the mocked http endpoint, form element error messages

> :bulb: To compare the code changes between the tutorial steps,
> just go to https://github.com/inkognitro/react-app-tutorial-code/compare and choose the specific tutorial step
> branches to compare. Some steps are split into smaller branches.
> Every step is an accumulation of all the steps before plus the code changes of the tutorial step.
> 
> Feel free to use the code for your own project. It's available under the MIT license :wink:

[Let's start »](01-setup.md)