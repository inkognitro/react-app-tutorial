[« previous](04-i18n.md) | [next »](06-form.md)

## 5. Toaster
Within this chapter we are going to provide a way to fire messages which are displayed to the current user
on the website. This is useful for example to inform a user about a successful save process in the backend.

## 5.1 Creating the toaster package
It would be nice to have a service to easily trigger some toast messages with different severities like
`success` or `error`.  Best case is to be able to trigger a message containing any content: A `ReactNode` or string.
I think it's useful to let the messages disappear after 3 seconds or so.
So let's try to define an interface and a context for our needs:

```typescript jsx
// src/packages/core/toaster/toaster.ts

import { v4 } from 'uuid';
import { createContext, ReactNode, useContext } from 'react';

type Severity = 'info' | 'success' | 'warning' | 'error';

type ToastMessageContent = ReactNode | string;

export type ToastMessage = {
    id: string;
    severity: Severity;
    content: ReactNode | string;
    autoHideDurationInMs?: null | number;
};

export type ToastMessageCreationSettings = Partial<ToastMessage> & { content: ToastMessageContent };

export function createToastMessage(settings: ToastMessageCreationSettings): ToastMessage {
    return {
        id: v4(),
        severity: 'info',
        autoHideDurationInMs: 3000,
        ...settings,
    };
}

export interface Toaster {
    showMessage(settings: ToastMessageCreationSettings): void;
}

const toasterContext = createContext<null | Toaster>(null);
export const ToasterProvider = toasterContext.Provider;

export function useToaster(): Toaster {
    const ctx = useContext(toasterContext);
    if (!ctx) {
        throw new Error('no Toaster was provided');
    }
    return ctx;
}
```

I think from a toaster-user-perspective this is an acceptable solution.
But from a toaster-manager-perspective we need to consider that when- or even wherever a toast message is dispatched
the toast messages should run through a central bus and be forwarded to every handler which want to do something
with it.

> :bulb: The [observer pattern](https://www.tutorialspoint.com/design_pattern/observer_pattern.htm):
> In terms of the observer patter a "subject" normally has a list of "observers".
> These observers are triggered by the subject everytime a specific event happens.
> This event can be a state change or something other. One may call this also the "listener" or "subscriber" pattern.

So let's reach this with a subscribable toaster implementation:

```typescript jsx
// src/packages/core/toaster/subscribableToaster.ts

import { createContext, useContext } from 'react';
import { Toaster, ToastMessageCreationSettings, createToastMessage, ToastMessage } from './toaster';

export type ToasterSubscriber = {
    id: string;
    onShowMessage: (message: ToastMessage) => void;
};

export class SubscribableToaster implements Toaster {
    private subscribers: ToasterSubscriber[];

    constructor() {
        this.subscribers = [];
        this.showMessage = this.showMessage.bind(this);
        this.subscribe = this.subscribe.bind(this);
        this.unSubscribe = this.unSubscribe.bind(this);
    }

    showMessage(settings: ToastMessageCreationSettings) {
        const toastMessage = createToastMessage(settings);
        this.subscribers.forEach((subscriber) => {
            subscriber.onShowMessage(toastMessage);
        });
    }

    subscribe(subscriber: ToasterSubscriber) {
        this.subscribers = [...this.subscribers, subscriber];
    }

    unSubscribe(subscriberId: string) {
        this.subscribers = this.subscribers.filter((subscriber) => subscriber.id !== subscriberId);
    }
}

const subscribableToasterContext = createContext<null | SubscribableToaster>(null);
export const SubscribableToasterProvider = subscribableToasterContext.Provider;

export function useSubscribableToaster(): SubscribableToaster {
    const toaster = useContext(subscribableToasterContext);
    if (!toaster) {
        throw new Error('no SubscribableToaster was provided');
    }
    return toaster;
}
```

And finally, export the files like so:

```typescript jsx
// src/packages/core/toaster/index.ts

export * from './toaster';
export * from './subscribableToaster';
```

## 5.2 Providing the services
Because the toaster implementations are not really dependent on the environment.
We can use the same logic for the `<TestServiceProvider>` as we are going to write in the
`<ServiceProvider>` component. Just add the following code to the two files

```typescript jsx
// src/ServiceProvider.tsx and src/TestServiceProvider.tsx

// add the following import statement:
import { ToasterProvider, SubscribableToaster, SubscribableToasterProvider } from '@packages/core/toaster';

// create a toaster reference in the top of the service provider component
const toasterRef = useRef(new SubscribableToaster());

// provide the two toaster context values in the return statement:
return (
    <ToasterProvider value={toasterRef.current}>
        <SubscribableToasterProvider value={toasterRef.current}>
            {/* the already existing context providers... */}
        </SubscribableToasterProvider>
    </ToasterProvider>
```
In the meantime absolutely known stuff for us.
Let's create a toast message renderer in the next step.

## 5.3 Designing a toaster subscriber with MUI
Next we need to create a subscriber or namely "observer" for our provided `SubscribableToaster`.
So let's try to create a component which subscribes to the `SubscribableToaster` when it is mounted,
but also unsubscribes from the list when it unmounts:

```typescript jsx
// src/packages/core/toaster/MuiToasterSubscriber.tsx

import React, { FC, useEffect, useRef, useState } from 'react';
import { Fade, Alert, Snackbar } from '@mui/material';
import { ToastMessage } from './toaster';
import { v4 } from 'uuid';
import { SubscribableToaster } from './subscribableToaster';

type MuiToastMessageProps = {
    data: ToastMessage;
    onClose: () => void;
};

const snackbarCloseAnimationDurationInMs = 300;

const MuiToastMessage: FC<MuiToastMessageProps> = (props) => {
    const [open, setOpen] = useState(true);
    function triggerCloseOnParentAfterCloseAnimationHasFinished() {
        setOpen(false);
        setTimeout(() => props.onClose(), snackbarCloseAnimationDurationInMs);
    }
    return (
        <Snackbar
            anchorOrigin={{ vertical: 'top', horizontal: 'center' }}
            autoHideDuration={props.data.autoHideDurationInMs}
            TransitionComponent={Fade}
            open={open}
            onClose={() => triggerCloseOnParentAfterCloseAnimationHasFinished()}>
            <Alert
                variant="filled"
                severity={props.data.severity}
                onClose={() => triggerCloseOnParentAfterCloseAnimationHasFinished()}>
                {props.data.content}
            </Alert>
        </Snackbar>
    );
};

export type MuiToasterSubscriberProps = {
    toaster: SubscribableToaster;
};

export const MuiToasterSubscriber: FC<MuiToasterSubscriberProps> = (props) => {
    const subscriberIdRef = useRef(v4());
    const pipelinedMessagesRef = useRef<ToastMessage[]>([]);
    const activeMessageRef = useRef<null | ToastMessage>(null);
    const [activeMessage, setActiveMessage] = useState<null | ToastMessage>(null);
    function showNextMessage() {
        const nextMessage = pipelinedMessagesRef.current.shift();
        if (activeMessageRef.current && !nextMessage) {
            activeMessageRef.current = null;
            setActiveMessage(null);
            return;
        }
        if (!nextMessage) {
            return;
        }
        activeMessageRef.current = nextMessage;
        setActiveMessage(nextMessage);
    }
    useEffect(() => {
        props.toaster.subscribe({
            id: subscriberIdRef.current,
            onShowMessage: (message: ToastMessage) => {
                pipelinedMessagesRef.current.push(message);
                if (!activeMessageRef.current) {
                    showNextMessage();
                }
            },
        });
        return () => props.toaster.unSubscribe(subscriberIdRef.current);
    }, [props.toaster, subscriberIdRef, pipelinedMessagesRef, activeMessageRef, showNextMessage]);
    if (!activeMessage) {
        return null;
    }
    return <MuiToastMessage key={activeMessage.id} data={activeMessage} onClose={() => showNextMessage()} />;
};
```
Add this component also to the exports in the `index.ts`.

> :bulb: The return statement of a hook must either be `undefined` or a function.
> The return function is a cleanup function.
> It's the equivalent of the class components `componentDidUnmount` with the possibility of better encapsulation.

## 5.4 Implementing the toaster subscriber
I think the `BlankPage` component is the right place to implement the toaster subscriber we've created.
With the following additions, we can support toast messages for all pages:

```typescript jsx
// src/components/page-layout/BlankPage.tsx

// add the following import:
import { MuiToasterSubscriber, useSubscribableToaster } from '@packages/core/toaster';

// use the subscribable toaster in the top of the BlankPage component:
const toaster = useSubscribableToaster();

// make the return statement of the BlankPage component look like so:
return (
    <>
        <CssBaseline />
        {props.children}
        <MuiToasterSubscriber toaster={toaster} />
    </>
);
```

## 5.4. Dispatch some toast messages
To be able to test if our toaster works, we should add some toast-message-dispatcher-links.
This time we use the `useToaster()` hook because we only want to dispatch messages no matter which
implementation of the toaster is behind.

```typescript jsx
// src/pages/IndexPage.tsx

import { FC } from 'react';
import { NavBarPage } from '@components/page-layout';
import { T, useTranslator } from '@packages/core/i18n';
import { useCurrentUser } from '@packages/core/auth';
import { FunctionalLink } from '@packages/core/routing';
import { useToaster } from '@packages/core/toaster';
import { Alert } from '@mui/material';

export const IndexPage: FC = () => {
    const { t } = useTranslator();
    const currentUser = useCurrentUser();
    const { showMessage } = useToaster();
    const username =
        currentUser.type === 'authenticated' ? currentUser.data.username : t('core.currentUser.guestDisplayName');
    const greeting = <T id="pages.indexPage.greeting" placeholders={{ username: <strong>{username}</strong> }} />;
    return (
        <NavBarPage title="Home">
            {greeting}
            <div style={{ marginTop: '15px' }}>
                <Alert severity="info">
                    <strong>MuiToasterSubscriber:</strong>
                    <br />
                    Note that if a toast message is displayed and you click outside of it, this toast message will
                    automatically be closed.
                    <br />
                    <br />
                    <FunctionalLink onClick={() => showMessage({ content: greeting })}>
                        trigger info toast
                    </FunctionalLink>
                    <br />
                    <FunctionalLink
                        onClick={() => {
                            showMessage({
                                severity: 'success',
                                autoHideDurationInMs: 1000,
                                content: <>First: {greeting}</>,
                            });
                            showMessage({
                                severity: 'success',
                                autoHideDurationInMs: 1000,
                                content: <>Second: {greeting}</>,
                            });
                        }}>
                        trigger multiple success toasts
                    </FunctionalLink>
                </Alert>
            </div>
        </NavBarPage>
    );
};
```

:floppy_disk: [branch 05-toaster](https://github.com/inkognitro/react-app-tutorial-code/compare/04-i18n-2...05-toaster)

[« previous](04-i18n.md) | [next »](06-form.md)