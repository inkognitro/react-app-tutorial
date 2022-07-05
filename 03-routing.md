[« previous](02-authentication.md) | [next »](04-i18n.md)

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
This makes sure, that the context is initialized only at the first render of the component.

```typescript jsx
import { Config, ConfigProvider } from '@packages/core/config';
```

Then add this lines before the return statement of the provider:

```typescript jsx
const configRef = useRef<Config>({
    companyName: 'ACME',
});
```

Finally, like for the current user, we need to provide the config by wrapping the `<App>`
with the `<ConfigProvider>`:

```typescript jsx
return (
    <ConfigProvider value={configRef.current}>
        // previous returned stuff...
    </ConfigProvider>
);
```

Well done! We are going to use this context later in this chapter.

### 3.4 Installing a routing library
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
Before we are going to create a page component, we need a reusable page layout.
This ensures that we don't have to write the same logic for every single page over and over again.
I think a new `BlankPage` component should build the base for every page layout.

```typescript jsx
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

Next, let's create a `NavBarPage` component based on the `BlankPage`:

```typescript jsx
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
and the export file:

```typescript jsx
// src/components/page-layout/index.ts

export * from './BlankPage';
export * from './NavBarPage';
```

### 3.5 Create the page components 
Before we create some routes we need to prepare route targets.
So let's create the page components of this tutorial.

The index page:
```typescript jsx
// src/pages/IndexPage.tsx

import { FC } from 'react';
import { NavBarPage } from '@components/page-layout';

export const IndexPage: FC = () => {
    return <NavBarPage title="Home">Home.</NavBarPage>;
};
```

The `RegisterPage`, within we are going to build the user registration form:
```typescript jsx
// src/pages/auth/RegisterPage.tsx

import { FC } from 'react';
import { NavBarPage } from '@components/page-layout';

export const RegisterPage: FC = () => {
    return <NavBarPage title="Register">Register.</NavBarPage>;
};
```

A `MySettingsPage` to demonstrate what happens if the user logs out while being on this page:
```typescript jsx
// src/pages/user-management/MySettingsPage.tsx

import { FC } from 'react';
import { NavBarPage } from '@components/page-layout';

export const MySettingsPage: FC = () => {
    return <NavBarPage title="My settings">My settings.</NavBarPage>;
};
```

And finally, a `NotFoundPage`. Keep in mind that the server only sends the `index.html`
and the bundled app in form of a `.js` file, so the routing happens on client side!
However, you won't be able to send a 404 status code from the server side,
until you do or at least a server side check before sending the http header to the client.
There are various ways to solve this, but this requires a server side script, which we do not cover in this tutorial.
```typescript jsx
// src/pages/NotFoundPage.tsx

import { FC } from 'react';
import { NavBarPage } from '@components/page-layout';

export const NotFoundPage: FC = () => {
    return <NavBarPage title="Not Found">Not Found.</NavBarPage>;
};
```

### 3.6 Time to wire the parts together
We still have no routing entry point.
As we've learned in the chapters before, not only we have to find a way to support routing in the browser
but also for the testing environment. So let's implement the specific routing in the right parts of our code.

Let's first wrap our `<App>` with the `<BrowserRouter>` within the service provider for the browser environment:
```typescript jsx
// src/ServiceProvider.tsx

import React, { FC, PropsWithChildren, useRef, useState } from 'react';
import {
    anonymousAuthUser,
    AuthUser,
    BrowserCurrentUserRepository,
    CurrentUserProvider,
    CurrentUserRepositoryProvider,
} from '@packages/core/auth';
import { Config, ConfigProvider } from '@packages/core/config';
import { BrowserRouter } from 'react-router-dom';

export const ServiceProvider: FC<PropsWithChildren<{}>> = (props) => {
    const [currentUserState, setCurrentUserState] = useState<AuthUser>(anonymousAuthUser);
    const browserCurrentUserRepositoryRef = useRef(new BrowserCurrentUserRepository(setCurrentUserState));
    const configRef = useRef<Config>({
        companyName: 'ACME',
    });
    return (
        <BrowserRouter>
            <ConfigProvider value={configRef.current}>
                <CurrentUserRepositoryProvider value={browserCurrentUserRepositoryRef.current}>
                    <CurrentUserProvider value={currentUserState}>
                        {props.children}
                    </CurrentUserProvider>
                </CurrentUserRepositoryProvider>
            </ConfigProvider>
        </BrowserRouter>
    );
};
```

The same applies to the testing environment's service provider but with another implementation.
Luckily react-router-dom provides the `MemoryRouter` for us:
```typescript jsx
// src/TestServiceProvider.tsx

import {
    anonymousAuthUser,
    AuthUser,
    CurrentUserProvider,
    CurrentUserRepository,
    CurrentUserRepositoryProvider,
} from '@packages/core/auth';
import React, { FC, PropsWithChildren, useRef } from 'react';
import { Config, ConfigProvider } from '@packages/core/config';
import { MemoryRouter } from 'react-router-dom';

class StubCurrentUserRepository implements CurrentUserRepository {
    setCurrentUser(currentUser: AuthUser) {}
    init() {}
}

export const TestServiceProvider: FC<PropsWithChildren<{}>> = (props) => {
    const stubCurrentUserRepositoryRef = useRef(new StubCurrentUserRepository());
    const configRef = useRef<Config>({
        companyName: 'ACME',
    });
    return (
        <MemoryRouter>
            <ConfigProvider value={configRef.current}>
                <CurrentUserRepositoryProvider value={stubCurrentUserRepositoryRef.current}>
                    <CurrentUserProvider value={anonymousAuthUser}>
                        {props.children}
                    </CurrentUserProvider>
                </CurrentUserRepositoryProvider>
            </ConfigProvider>
        </MemoryRouter>
    );
};
```

Finally, we are going to define our routes within the `<App>` component. Just delete the other stuff
we have extracted to our `<NavBarPage>` component, so that the `App.tsx` looks like below:
```typescript jsx
// src/App.tsx

import React, { useEffect } from 'react';
import { useCurrentUser, useCurrentUserRepository } from '@packages/core/auth';
import { Route, Routes } from 'react-router-dom';
import { IndexPage } from '@pages/IndexPage';
import { RegisterPage } from '@pages/auth/RegisterPage';
import { MySettingsPage } from '@pages/user-management/MySettingsPage';
import { NotFoundPage } from '@pages/NotFoundPage';

function AppRoutes() {
    const currentUser = useCurrentUser();
    const isUserLoggedIn = currentUser.type === 'authenticated';
    return (
        <Routes>
            <Route path="/" element={<IndexPage />} />
            <Route path="/auth/register" element={<RegisterPage />} />
            {isUserLoggedIn && <Route path="/user-management/my-settings" element={<MySettingsPage />} />}
            <Route path="*" element={<NotFoundPage />} />
        </Routes>
    );
}

function App() {
    const currentUserRepo = useCurrentUserRepository();
    useEffect(() => {
        currentUserRepo.init();
    }, [currentUserRepo]);
    return <AppRoutes />;
}

export default App;
```

Nice one! You should now be able to go to different routes in the app.
At [localhost:3000/foo](http://localhost:3000/foo) you should see the `NotFoundPage`.
This page should also appear, when you log in, switch to [localhost:3000/user-management/my-settings](http://localhost:3000/user-management/my-settings),
and then do a logout.

Also execute `npm run test` in your console and see if your test passes.
Otherwise, have a look at the [03-routing-1 branch](https://github.com/inkognitro/react-app-tutorial-code/compare/02-auth-2...03-routing-1)
to compare with yours.

:floppy_disk: [branch 03-routing-1](https://github.com/inkognitro/react-app-tutorial-code/compare/02-auth-2...03-routing-1)

### 3.7 Next design level with MUI
There's no need to reinvent the wheel over and over again by creating everything from scratch.
[Material UI (MUI)](https://mui.com/) offers a huge set of components which are easy to customize and well tested.
It's the preferred framework these days for most developers. It's compatibility with React makes it a perfect fit for us.
Add the library and the MUI icons with:

```
npm install @mui/material @mui/icons-material --save
```

Material UI comes with the [emotion](https://emotion.sh/docs/introduction) library by default for adding custom css.
These two libraries are almost the same.
In this tutorial, we go with [styled-components](https://styled-components.com/) instead,
to gain more experience about what is going on under the hood with webpack and TS.

To have a comparison of these two libraries, just have a look at
[this LogRocket article](https://blog.logrocket.com/styled-components-vs-emotion-for-handling-css/).
With [MUI](https://mui.com/) we need to switch its styled-engine to [styled-components](https://styled-components.com/).

Install the styled-components library and its [MUI](https://mui.com/) styled-engine.
```
npm install @mui/styled-engine-sc styled-components --save
```

We also need to install the TS types for it, because the styled-components does not support TS by default.

```
npm install @types/styled-components --save-dev
```

Furthermore and to make it work with Jest, we need to do a bit more than just described in the
[MUI documentation](https://mui.com/material-ui/guides/styled-engine/).
Don't worry, we'll go through it together:

1. Add `"@mui/styled-engine": ["./node_modules/@mui/styled-engine-sc"]` to `paths` property in `tsconfig.json`
2. Add the `'@mui/styled-engine': '@mui/styled-engine-sc'` alias in `config/webpack.config.js`
3. Add `"^@mui/styled-engine$": "<rootDir>/node_modules/@mui/styled-engine-sc"` in `moduleNameMapper` property of `package.json`

Not that hard.

### 3.8 Integrate the web fonts for MUI
In order to fully install MUI, we need to provide the required fonts to the user's browser.
In general, I prefer to download the fonts from external sources
and to directly provide them to the user from my own server.
This allows us to remain independent of third-party providers.
This is also desirable from a security and availability perspective.
Nevertheless, let's include the fonts directly from Google for now and add the following
parts in the `<head>` of the `public/index.html` file by adding the following lines:

```html
<link rel="stylesheet" href="https://fonts.googleapis.com/css?family=Roboto:300,400,500,700&display=swap" />
<link rel="stylesheet" href="https://fonts.googleapis.com/icon?family=Material+Icons" />
```

### 3.9 Clean up the document body's css
First we should clean up the document `body`'s margin.
I don't know why but the default user agent stylesheet has a `margin: 8px;` setting.
Probably a leftover from earlier times which wasn't removed for not breaking existing websites or so.
Let's clean this up by adding a global style with [styled-components](https://styled-components.com/)
to our `BlankPage` component.

```typescript jsx
// src/components/page-layout/BlankPage.tsx

import React, { FC, ReactNode, useEffect } from 'react';
import { useConfig } from '@packages/core/config';
import { createGlobalStyle } from 'styled-components';

const GlobalStyle = createGlobalStyle`
  body {
    margin: 0;
  }
`;

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
            document.title = titleParts.join(' :: ');
        }
    });
    return (
        <>
            <GlobalStyle />
            {props.children}
        </>
    );
};
```

### 3.11 Provide the theme
To be able to access the theme we should create one and provide it like so:

```typescript jsx
// src/components/theme/mui.ts

import { createTheme } from '@mui/material';

export const theme = createTheme();
```
This is the place where we can modify the styles for our MUI components.

Let's export it like below.
```typescript jsx
// src/components/theme/index.ts

export * from './mui';
```

Finally, we should to provide the theme in the `ServiceProvider` and `TestServiceProvider` like so
```typescript jsx
// src/ServiceProvider.tsx and src/TestServiceProvider.tsx

// add the following import statements:
import { ThemeProvider as MuiThemeProvider } from '@mui/material';
import { ThemeProvider as ScThemeProvider } from 'styled-components';
import { theme } from '@components/theme';

// wrap the other providers with the theme service providers
return (
    <MuiThemeProvider theme={theme}>
        <ScThemeProvider theme={theme}>
            {/* other service providers */}
        </ScThemeProvider>
    </MuiThemeProvider>
);
```

### 3.10 Provide reusable link components
We should provide two different link components. Both should play well with the props of `@mui/material`'s link component.
One link component should have the functionality of `react-router-dom`'s `Link` to route to another url.
With the other link component it should be possible to only execute stuff by the `onClick` property without routing.
We should create these components in the `@packages/core` folder, to make it available for any other package or component
we have to create in the future.

```typescript jsx
// src/packages/core/routing/Link.tsx

import React, { FC, ReactNode } from 'react';
import { Link as MuiLink, LinkProps as MuiLinkProps } from '@mui/material';
import { Link as ReactRouterDomLink } from 'react-router-dom';

export type RoutingLinkProps = MuiLinkProps & {
    to: string;
    children?: ReactNode;
};

export const RoutingLink: FC<RoutingLinkProps> = (props) => {
    return <MuiLink {...props} component={ReactRouterDomLink} />;
};

export type FunctionalLinkProps = MuiLinkProps & {
    onClick: () => void;
};

export const FunctionalLink: FC<FunctionalLinkProps> = (props) => (
    <MuiLink
        {...props}
        href="#"
        onClick={(event) => {
            event.preventDefault();
            if (props.onClick) {
                props.onClick();
            }
        }}
    />
);
```

As always, we leak our public available components of the package with an `index.ts` file:
```typescript
// src/packages/core/routing/index.ts

export * from './Link';
```

### 3.11 Prevent our navigation from causing eye cancer
Now that we have MUI installed, we can use its components and provide a better
nav bar with no effort.

So let's change our `NavBarPage.tsx`, that it looks like so:

```typescript jsx
// src/components/page-layout/NavBarPage.tsx

import React, { FC, useState } from 'react';
import { BlankPage, BlankPageProps } from './BlankPage';
import { anonymousAuthUser, useCurrentUser, useCurrentUserRepository } from '@packages/core/auth';
import { useNavigate } from 'react-router-dom';
import { Button, Toolbar, Typography, Container, Menu, MenuItem } from '@mui/material';
import { useConfig } from '@packages/core/config';
import { Home } from '@mui/icons-material';
import { FunctionalLink, RoutingLink } from '@packages/core/routing';

const LoggedInUserMenu: FC = () => {
    const navigate = useNavigate();
    const currentUserRepo = useCurrentUserRepository();
    const currentUser = useCurrentUser();
    const [anchorEl, setAnchorEl] = useState<null | HTMLElement>(null);
    if (currentUser.type !== 'authenticated') {
        return null;
    }
    function logoutUser() {
        currentUserRepo.setCurrentUser(anonymousAuthUser);
        navigate('/');
    }
    function handleClick(event: React.MouseEvent<HTMLButtonElement>) {
        setAnchorEl(event.currentTarget);
    }
    function closeMenu() {
        setAnchorEl(null);
    }
    const isMenuOpen = !!anchorEl;
    return (
        <>
            <Button
                id="basic-button"
                aria-controls={isMenuOpen ? 'basic-menu' : undefined}
                aria-haspopup="true"
                aria-expanded={isMenuOpen ? 'true' : undefined}
                onClick={handleClick}>
                {currentUser.data.username}
            </Button>
            <Menu
                id="basic-menu"
                anchorEl={anchorEl}
                open={isMenuOpen}
                onClose={closeMenu}
                MenuListProps={{ 'aria-labelledby': 'basic-button' }}>
                <MenuItem
                    onClick={() => {
                        navigate('/user-management/my-settings');
                        closeMenu();
                    }}>
                    My settings
                </MenuItem>
                <MenuItem
                    onClick={() => {
                        logoutUser();
                        closeMenu();
                    }}>
                    Logout
                </MenuItem>
            </Menu>
        </>
    );
};

const Nav: FC = () => {
    const { companyName } = useConfig();
    const navigate = useNavigate();
    const currentUserRepo = useCurrentUserRepository();
    const currentUser = useCurrentUser();
    const isLoggedIn = currentUser.type === 'authenticated';
    function loginUser() {
        currentUserRepo.setCurrentUser({
            type: 'authenticated',
            apiKey: 'foo',
            data: {
                id: 'foo',
                username: 'Linus',
            },
        });
    }
    return (
        <Toolbar sx={{ borderBottom: 1, borderColor: 'divider', marginBottom: '15px' }}>
            <RoutingLink to="/">
                <Home />
            </RoutingLink>
            <Typography component="h2" variant="h5" color="inherit" align="center" noWrap sx={{ flex: 1 }}>
                {companyName}
            </Typography>
            {!isLoggedIn && (
                <>
                    <FunctionalLink onClick={loginUser} noWrap variant="button" href="/" sx={{ p: 1, flexShrink: 0 }}>
                        Login
                    </FunctionalLink>{' '}
                    <Button variant="outlined" size="small" onClick={() => navigate('/auth/register')}>
                        Sign up
                    </Button>
                </>
            )}
            {isLoggedIn && <LoggedInUserMenu />}
        </Toolbar>
    );
};

export type NavBarPageProps = BlankPageProps;

export const NavBarPage: FC<NavBarPageProps> = (props) => {
    return (
        <BlankPage title={props.title}>
            <Nav />
            <Container>{props.children}</Container>
        </BlankPage>
    );
};
```

Well done! I think it's time to have a look at translating things in the next chapter.

:floppy_disk: [branch 03-routing-2](https://github.com/inkognitro/react-app-tutorial-code/compare/03-routing-1...03-routing-2)

[« previous](02-authentication.md) | [next »](04-i18n.md)