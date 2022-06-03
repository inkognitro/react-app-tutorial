> Special thanks go to [easylearn schweiz ag](https://easylearn.ch). This is the company I currently work for.
> Our managers allowed me to create parts of this tutorial at working time.

!!! WORK IN PROGRESS !!!

# react-app-tutorial
The purpose of this tutorial is to serve for a better understanding of the concepts of React and to know which libraries
are state of the art to handle daily frontend stuff.
To go through this tutorial without a lot of pain, you should be familiar with Javascript ES6+, React and JSX.
This tutorial is written in Typescript, but I think most people who understand Javascript can also learn
Typescript by doing this tutorial. No need to panic: It is just "javascript with type hints".

### What we are going to build together
Together we are going to build a single page app with an `index` and a `register user page`.
The user registration form is going to be wired with a mocked api over http to also cover network traffic.
The output of this tutorial is a clean and reusable framework built on top of react.

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
3. [Routing](03-routing.md) - Prepare page layouts and the page's skeletons, mui, styled-components
4. [Internationalization](04-i18n.md) - Translate contents, localization base, switching between two languages
5. [Toaster](05-toaster.md) - Provide a way to send push messages (toasts) to the website user (WIP)

> :bulb: To compare the code changes between the tutorial steps,
> just go to https://github.com/inkognitro/react-app-tutorial-code/compare and choose the specific tutorial step
> branches to compare. Some steps are split into smaller branches.
> Every step is an accumulation of all the steps before plus the code changes of the tutorial step.
> 
> Feel free to use the code for your own project. It's available under the MIT license :neutral_face:

[Let's start »](01-setup.md)

## Feedbacks
[Sam](https://github.com/sami-akkawi):
- **Chapter 3:** Maybe explain a bit about the external components that we are using, like: Context.Provider, Typography/Toolbar and their props like component, variant, sx

## Planned Tutorial steps for now
This will probably change during the creation of the tutorial.

1. Setup react app from scratch (**done**)
   - installation with create-react-app
   - linting
2. Auth (**done**)
   - packages folder
   - define current user types
   - define `useCurrentUser` hook
   - define `CurrentUserRepository` and its hook
   - adjust navigation (add username when current user is logged in)
3. Routing (**done**)
   - routing with react-router-dom
   - install material ui
   - page layout
   - navigation
   - index page
   - register user page
4. Translator, i18n (**done**)
   - setup translation with `i18next`
   - `useTranslator` hook
   - `Translation` component with `ReactNode` placeholders
   - setup context
5. Toaster (**in progress**)
   - **Utils**: `Message` type and `Message(s)` component
   - define toaster types
   - define `SubscribableToaster` class
   - setup context
   - create toaster with MUI
   - implement toaster component in root
   - create a link to dispatch a toast at index page
6. Collections and Providers
   - define types
   - createArrayProvider
   - explanation that this should also be done for `api-v1` collections
7. Form elements
   - form element type definitions
   - `TextField`, `Dropdown` (use `Messages` component from utilities)
   - Error message enrichment by `modifier` function
   - Provide the user registration form skeleton
8. ApiV1
   - create [axios](https://axios-http.com) `http` request handler
   - create ApiV1RequestHandler
   - define `registerUserEndpoint.ts` with `shouldFail: boolean` param
   - build the `useApiV1RequestHandler` hook, open for middleware
9. Wire the User registration form
   - use the `useApiV1RequestHandler` hook
   - make two buttons `registerAndSucceed` and `registerAndFail` 
   - extend the `useApiV1RequestHandler` hook with `showToastMessages: boolean` and its middleware
10. Summing up
    - What to do next: Optimize with [NextJs](https://nextjs.org)?
    - Some differences to current architecture in naming (Provided Vs. Provider)