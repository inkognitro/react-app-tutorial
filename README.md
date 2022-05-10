# react-app-tutorial
LICENCE: 2022 MIT, Marcel Fischer

## Introduction (this file):
Preconditions:
    - Javascript
    - JSX (react)
    - Typescript

## Tutorial

### What we are going to build
We are going to build a single page app with an `index` and a `register user page`.
The user registration form is going to be wired with a mocked api over http (mocked api is written in Go).
Redux is out of scope of this tutorial.

### Steps
1. Setup react app from scratch
   - installation with create-react-app
   - code structure (packages/core, spa) and the "whys" (take explanations from coding guidelines)
   - routing with react-router-dom
   - install material ui
   - install styled-components (reference VueJs strategy)
2. Pages skeletons
   - page layout
   - navigation
   - index page
   - register user page
3. Translator (i18n)
   - setup translation with `i18next`
   - `useTranslator` hook
   - `Translation` component with `ReactNode` placeholders
   - setup context
4. Utilities
   - `Message` type and `Message(s)` component
   - JWT (Json Web Token), [jwt.io](https://jwt.io)
5. Toaster
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
   - build the `useApiV1RequestHandler` hook (without toaster link)
9. Wire the User registration form
   - use the `useApiV1RequestHandler` hook
   - make two buttons `registerAndSucceed` and `registerAndFail` 
   - extend the `useApiV1RequestHandler` hook with `showToastMessages: boolean`
9. Summing up
   - What to do next: Optimize with [NextJs](https://nextjs.org)?
   - Some differences to current architecture in naming (Provided Vs. Provider)