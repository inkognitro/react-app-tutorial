[« previous](02-authentication.md) | [next »](04-foo.md)

## 3. Routing 
Within this chapter we go through routing stuff and create the page layout base.

### 3.1 Add two more aliases
First, we should create two more aliases:
A `@components` alias for the new `src/component/` folder where our general app specific components live in,
and a `@pages` alias for the new `src/pages` folder where our page components will be defined.
The procedure is the same as described in [`2.7 Ramp up our import paths`](02-authentication.md#27-ramp-up-our-import-paths).
I am sure you can manage to get that done on your own :sunglasses:

### 3.2 Add a config context
To query some general configurations of the app I recommend centralizing those in a separate context.
For the next step of our tutorial it is required to have access to the company name of our app.
The procedure is the same as we've done for our current user context.
So let's add a new config context with following files.

```typescript
// src/packages/core/config/config.ts

import { createContext, useContext } from 'react';

export type Config = {
    companyName: string;
};

const configContext = createContext<Config | null>(null);

export const ConfigProvider = configContext.Provider;

export function useConfig(): Config {
    const config = useContext(configContext);
    if (!config) {
        throw new Error(`no config was provided`);
    }
    return config;
}
```

and as usual, with the export from an index file:

```typescript
// src/packages/core/config/index.ts

export * from './config';
```

However, this context needs to be integrated as well.
Add a new context reference to `src/ServiceProvider.tsx` and `src/TestServiceProvider.tsx`.
This can be done with the `React.useRef` hook.
This makes sure, that the context is initialized at the first render of the component.

```typescript
import { Config, ConfigProvider } from '@packages/core/config';
```

Then add this lines before the return statement of the provider:

```typescript
const configRef = useRef<Config>({
    companyName: 'ACME',
});
```

Finally and like for the current user, you need to provide the config by wrapping your app
with its provider:

```typescript
return (
    <ConfigProvider value={configRef.current}>
        // previous returned stuff...
    </ConfigProvider>
);
```

Well done! We are going to use this context later in this chapter.

### 3.4 Install a routing library
Every browser app with multiple pages requires some routing mechanism.
In case of React the most common library to have in mind is [react-router](https://reactrouter.com).
We could build our own routing logic, but I think for such a common task it absolutely makes sense
to take a proven solution which goes hand in hand with the browser history.
Nice to know: [react-router](https://reactrouter.com) does also provide a router for [react-native](https://reactnative.dev).

Install it with:
```
npm install react-router-dom --save 
```

We are going to use the `<Link>` component of it in the next step.

### 3.5 Page layout components
Before we create a page component, we need a reusable page layout.
This ensures that we don't have to write the same logic whenever we create a page component.
A new `BlankPage` component should build our base for every page layout.

```typescript
// src/components/page-layout/BlankPage.tsx

import React, { FC, ReactNode, useEffect } from 'react';
import { useConfig } from '@packages/core/config';

export type BlankPageProps = {
    title: string;
    children?: ReactNode;
};

export const BlankPage: FC<BlankPageProps> = (props) => {
    const { companyName } = useConfig();
    const titleParts: string[] = [];
    if (props.title) {
        titleParts.push(props.title);
    }
    if (companyName) {
        titleParts.push(companyName);
    }
    useEffect(() => {
        if (document) {
            // only adjust this in browser environment
            document.title = titleParts.join(' :: ');
        }
    });
    return <>{props.children}</>;
};
```

Let's create a `NavBarPage` component based on the `BlankPage`:

```typescript
// src/components/page-layout/NavBarPage.tsx

import React, { FC, MouseEvent } from 'react';
import { BlankPage, BlankPageProps } from './BlankPage';
import { anonymousAuthUser, useCurrentUser, useCurrentUserRepository } from '@packages/core/auth';
import { Link, useNavigate } from 'react-router-dom';

function Nav() {
    const navigate = useNavigate();
    const currentUserRepo = useCurrentUserRepository();
    const currentUser = useCurrentUser();
    const isLoggedIn = currentUser.type === 'authenticated';
    const currentUserDisplayName = currentUser.type === 'authenticated' ? currentUser.data.username : 'Anonymous';
    function loginUser(event: MouseEvent<HTMLAnchorElement>) {
        event.preventDefault();
        currentUserRepo.setCurrentUser({
            type: 'authenticated',
            apiKey: 'foo',
            data: {
                id: 'foo',
                username: 'Linus',
            },
        });
    }
    function logoutUser(event: MouseEvent<HTMLAnchorElement>) {
        event.preventDefault();
        currentUserRepo.setCurrentUser(anonymousAuthUser);
        navigate('/');
    }
    return (
        <div
            style={{
                marginLeft: 'auto',
                marginRight: 'auto',
                width: '600px',
                textAlign: 'center',
            }}>
            <Link to="/">Home</Link> &ndash;{' '}
            {!isLoggedIn && (
                <>
                    <Link to="/auth/register">Register</Link> &ndash;{' '}
                    <a href="#" onClick={loginUser}>
                        Login
                    </a>
                </>
            )}
            {isLoggedIn && (
                <a href="#" onClick={logoutUser}>
                    Logout
                </a>
            )}{' '}
            :: {isLoggedIn && <Link to="/user-management/my-settings">{currentUserDisplayName}</Link>}
            {!isLoggedIn && currentUserDisplayName}
        </div>
    );
}

export type NavBarPageProps = BlankPageProps;

export const NavBarPage: FC<NavBarPageProps> = (props) => {
    return (
        <BlankPage title={props.title}>
            <Nav />
            {props.children}
        </BlankPage>
    );
};
```

### 3.5 Create the page components


### 3.6 Time to wire our page components


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