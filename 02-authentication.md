[« previous](01-setup.md) | [next »](03-routing.md)

## 2. Authentication
To provide a nice user experience we need to know who our app visitors are.
We need some sort of current user object and user cache mechanism.
We are going to build that in this chapter.

### 2.1 Separated `packages/` folder
Not only with testing in mind, but also in consideration of the principle of
[low coupling and high cohesion](https://stackoverflow.com/questions/14000762/what-does-low-in-coupling-and-high-in-cohesion-mean),
I think it makes sense to write the `auth` logic as independent as possible.
Best case scenario would be to be able to reuse the logic for another app.
This other app could be a smaller app for embedding some chosen contents in an iframe
which is rendered by an external website. Let's extract such logic into a separate `packages/` folder.

Sometimes the boundary between app logic and package logic can be really opaque.
In such a case I suggest creating the logic in the app's folder and extracting
your code into the `packages/` folder [only when you need it](https://martinfowler.com/bliki/Yagni.html)
at a later time.

I think our authentication and authorization logic is well accommodated in the `packages/core/auth` folder.

### 2.2 The auth user object
Whenever a user visits a website or an app, this user can either be identified as an anonymous or logged-in user.
The sentence before already implies, that we need two different types of a current user object.
One for an anonymous and one for a logged-in user.
So let's create these two with a generic, which is distinguishable by its `type` property.

> :bulb: The automatic type cast for a union type to one of its specific types in TS, can be reached with a unique value for the specific type's property defined in the generic type.

So let's add following files to the codebase:

```typescript
// packages/core/auth/authUser.ts

type AuthUserType = "anonymous" | "authenticated";
type GenericUser<T extends AuthUserType, Payload extends object = {}> = { type: T } & Payload;
type UserData = { id: string; username: string };
export type AnonymousAuthUser = GenericUser<"anonymous">;
export type AuthenticatedAuthUser = GenericUser<
    "authenticated",
    { apiKey: string; data: UserData }
>;
export type AuthUser = AnonymousAuthUser | AuthenticatedAuthUser;
```

### 2.3 DI, types and interfaces
We should never reference a service directly by its implementation,
otherwise you won't be able to properly test your components or other code you've written.
That's also true for React apps.
A good practise to prevent from directly coupled service implementations is the combination of
[dependency injection (DI)](https://stackify.com/dependency-injection/) with interfaces.
In React, we can reach DI with the [useContext hook](https://reactjs.org/docs/hooks-reference.html#usecontext).
At newer versions of TS, types and interfaces are nearly the same.
The only difference is that interfaces have the drawback to implicitly merge their definitions.
So let's go with types.

### 2.4 Current user and its repository hooks
After we defined the type `AuthUser` for a website visitor, we need a custom `useCurrentUser` hook
to reach the current user in every component no matter how nested it is.
We also need a repository (a service) to keep the current user after a page refresh has been done.
We should be able to mock/stub this service for testing purposes.

```typescript
// packages/core/auth/currentUser.ts

import { AnonymousAuthUser, AuthUser } from "./authUser";
import { createContext, useContext } from "react";

const currentUserContext = createContext<null | AuthUser>(null);

export const CurrentUserProvider = currentUserContext.Provider;

export const anonymousAuthUser: AnonymousAuthUser = { type: "anonymous" };

export function useCurrentUser(): AuthUser {
    const currentUser = useContext(currentUserContext);
    if (!currentUser) {
        throw new Error(`no AuthUser was provided`);
    }
    return currentUser;
}
```
and

```typescript
// packages/core/auth/currentUserRepository.ts

import { createContext, useContext } from "react";
import { AuthUser } from "./authUser";
import { anonymousAuthUser } from "./currentUser";

export type CurrentUserRepository = {
    setCurrentUser(currentUser: AuthUser): void;
    init: () => void;
};

type CurrentUserStateSetter = (currentUser: AuthUser) => void;

export class BrowserCurrentUserRepository implements CurrentUserRepository {
    private readonly setCurrentUserState: CurrentUserStateSetter;

    constructor(setCurrentUserState: CurrentUserStateSetter) {
        this.setCurrentUserState = setCurrentUserState;
    }

    setCurrentUser(currentUser: AuthUser) {
        this.setCurrentUserState(currentUser);
        if (currentUser.type === "anonymous") {
            localStorage.removeItem("currentUser");
            return;
        }
        localStorage.setItem("currentUser", JSON.stringify(currentUser));
    }

    init() {
        const currentUserStr = localStorage.getItem("currentUser");
        if (!currentUserStr) {
            this.setCurrentUser(anonymousAuthUser);
            return;
        }
        const currentUser = JSON.parse(currentUserStr) as AuthUser;
        this.setCurrentUser(currentUser);
    }
}

const currentUserRepositoryContext = createContext<CurrentUserRepository | null>(null);
export const CurrentUserRepositoryProvider = currentUserRepositoryContext.Provider;

export function useCurrentUserRepository(): CurrentUserRepository {
    const repo = useContext(currentUserRepositoryContext);
    if (!repo) {
        throw new Error(`no CurrentUserRepository was provided`);
    }
    return repo;
}
```
In Node, it is usual to provide a package's public API by an index file.
So to properly finalize our `auth` package we also add this clue code.

```typescript
// packages/core/auth/currentUserRepository.ts
export * from "./authUser";
export * from "./currentUser";
export * from "./currentUserRepository";
```
Well done! We've created our first independent package code.
Let's use it in our app now!

### 2.5 Using the auth package
tbd...

[« previous](01-setup.md) | [next »](03-routing.md)